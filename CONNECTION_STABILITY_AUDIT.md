# Connection-Stability Audit: Telnyx WebRTC iOS SDK (fork)

## Overview

Documentation-only audit of call-connection and reconnection stability in the
`justinwhytcross/telnyx-webrtc-ios` fork. **No SDK source, demo-app, podspec, or
verto-JSON files were modified.**

**Audited tree:** `feature/manual-audio-mode` — the branch the Rewynd app actually
ships (`TelnyxRTC.podspec:5` = `3.2.3-fork`). This is the fork's reason for existing:
SDK `3.2.0` + cherry-pick of [#337](https://github.com/team-telnyx/telnyx-webrtc-ios/pull/337)
+ the `manualAudioSessionManagement` patch. Every `file:line` below was verified against
this branch (see **Appendix A**). All cited code exists; no findings were dropped.

## Headline

**The fork is three minor versions behind upstream, and the single most impactful bug
in this audit is already fixed upstream.** Upstream `team-telnyx/telnyx-webrtc-ios` is at
`4.0.2` (+ an `[Unreleased]` fix). The HIGH connect-timeout defect that most plausibly
explains observed connection-delay complaints is resolved by upstream PR
[#348](https://github.com/team-telnyx/telnyx-webrtc-ios/pull/348). **The primary
recommendation is to merge upstream `4.0.2` + #348 and re-apply the
`manualAudioSessionManagement` patch** — estimated **medium** conflict cost (two files).
Everything else in this report is either inherited by that merge or a smaller
fork-specific follow-up.

Severity uses observation-grounded labels, not abstract HIGH/MED/LOW:

- **Observed** — a mechanism with a defensible link to a real user report.
- **Potential** — the mechanism exists in code but has not been tied to a specific report.
- **Documentation** — informational; no defect.

## 0. User complaints this audit addresses

Three findings are tied to real reports from the parent Rewynd context. Where a link is
mechanistically plausible but unproven without device logs, it is marked **Potential**
and the finding's severity is capped accordingly.

| User report | Symptom | Finding | Link strength |
|-------------|---------|---------|---------------|
| Tom Clark — calls `#4350008`, `#4290168` | Caller-side connection delay / abandonment | [Finding 1 — connect-timeout](#finding-1-observed--no-socket-connect-timeout-on-the-default-auto-region) | **Observed.** Corroborated by upstream #348's own changelog, which attributes hung dials to this exact code path. |
| Tom Clark — calls `#4350008`, `#4290168` | Slow time-to-ready before the call can proceed | [Finding 4 — gateway-registration ceiling](#finding-4-potential--gateway-registration-ceiling--9s) | **Potential.** `~9s` worst-case register time contributes to dial latency, but not isolated in logs. |
| Alex Canavan — call `4290007` | Cold-start dead-air (call connects, no audio) | [Finding 2 — reconnect ordering](#finding-2-potential--reconnectclient-arms-the-60s-reconnect-timer-before-validating-reconnect-is-possible) / [Finding 3 — reconnect thrash](#finding-3-potential--reconnect-thrash-on-network-flapping--no-debounce-no-in-flight-guard) | **Potential.** If `NetworkMonitor` fires a transition during cold launch while the pushed call is active, `reconnectClient()` runs `endForAttachCall()` + `disconnect(reconnect:)`, which can tear down audio on the freshly-connected call. Plausible but unproven without logs; cold-start dead-air may instead be an audio-session-activation issue in the `manualAudioSessionManagement` patch (app-side). |

Dead-air specifically is *call-connected-but-silent*, not *call-failed-to-connect*, so it
is **not** attributed to Finding 1 (which is a failed/hung-connect mechanism). Findings
with no observed link are labelled **Potential** below.

## 1. Executive summary

| # | Label | Finding | Location (`feature/manual-audio-mode`) | Impact |
|---|-------|---------|----------------------------------------|--------|
| 1 | **Observed** | No socket connect-timeout on default `.auto` region | `Services/Socket.swift:50`, `:86-93` | Hung TCP/TLS connect blocks up to `~120s`; ties to connection-delay/abandonment. **Fixed upstream (#348).** |
| 2 | Potential | `reconnectClient()` arms `60s` timer before validating reconnect is possible | `TxClient.swift:1612-1628` | Missing current call → state stuck `RECONNECTING` for `60s` → `reconnectFailed` |
| 3 | Potential | Reconnect thrash on network flapping — no debounce / in-flight guard | `TxClient.swift:329-339`, `:1627-1628` | Wi-Fi↔cellular handover fires repeated `disconnect(reconnect:)`; possible cold-start audio teardown |
| 4 | Potential | Gateway-registration ceiling `≈ 9s` | `TxClient.swift:130-131`, `:1100-1122` | Worst-case time-to-ready `~9s`; `gatewayNotRegistered` on exhaustion |
| 5 | Potential | VoIP-push INVITE timeout is a silent `10s` deadline | `TxClient.swift:195`, `:823-846` | No answer-flow retry; terminates with `ORIGINATOR_CANCEL`/`487` |
| 6 | Potential | Fixed reconnect cadence, no backoff | `TxClient.swift:133`, `:1724` | Flat `~1s` delay → retries every `~1s` for `60s` under sustained loss |
| 7 | Documentation | Dead connect-timeout assignment | `Services/Socket.swift:41`, `:47` | `5s` `connectionTimeout` overwritten by `TimeInterval(120)`; `5s` never used. **Fixed upstream (#348).** |
| 8 | Documentation | `checkInternetAccess` reaches `https://www.google.com` | `Utils/NetworkMonitor.swift:76` | Privacy/footprint note; VPN-classified paths only |

## 2. Findings — call connection & timeframe

### Finding 1 [Observed] — No socket connect-timeout on the default `.auto` region

`Socket.connect()` only arms `connectionTimeoutTimer` when `shouldFallbackToAuto` returns
`true`:

- `TelnyxRTC/Telnyx/Services/Socket.swift:50` — `if shouldFallbackToAuto(signalingServer: signalingServer) { startConnectionTimeout() }`
- Default region is `.auto` (`TelnyxRTC/Telnyx/Models/TxServerConfiguration.swift:29`, `region: Region = Region.auto`).
- Default host is `wss://rtc.telnyx.com` (`TelnyxRTC/Telnyx/InternalConfig.swift:13`, `prodHost`).
- `extractRegionPrefix(from:)` returns the first host component, so `rtc.telnyx.com` → `"rtc"` (`Socket.swift:95-102`).
- `Region(rawValue: "rtc")` is `nil`, so `shouldFallbackToAuto` returns `false` (`Socket.swift:86-93`).

On the default URL the timer **never starts**. A hung connect then falls back only to the
`URLRequest` timeout, which is overwritten to `TimeInterval(120)` (`Socket.swift:47`) — so
a lost SYN after a VoIP push can hang the dial for up to `~120s` with no SDK recovery.

**Observed link:** upstream #348's changelog describes this precise path — "a lost SYN
after a VoIP push left the dial hanging on TCP retransmit backoff with no recovery —
causing answer-from-push failures on cold start" — which matches Tom Clark's
connection-delay/abandonment reports (`#4350008`, `#4290168`).

**Resolution:** inherited by merging #348 (see §5). No fork-specific work required.

### Finding 5 [Potential] — VoIP-push INVITE timeout is a silent `10s` deadline

- `TelnyxRTC/Telnyx/TxClient.swift:195` — `INVITE_TIMEOUT_SECONDS: TimeInterval = 10.0`
- `startInviteTimeout()` schedules the timer at `:810` → `handleInviteTimeout()` at `:823-846`.
- On timeout the call terminates with `cause: "ORIGINATOR_CANCEL"` (`:828`), `causeCode: 487` (`:829`), `sipCode: 487` (`:830`); delegate end events fire; `answerCallAction?.fulfill()` is called. **No retry** of the answer flow.

**Recommendation:** expose as a `TxConfig.inviteTimeout` knob (code change).

### Finding 7 [Documentation] — Dead connect-timeout assignment

`TelnyxRTC/Telnyx/Services/Socket.swift:41` sets `request.timeoutInterval = connectionTimeout`
(`connectionTimeout = 5.0`, `Socket.swift:24`), immediately overwritten by
`TimeInterval(120)` at `:47`. The `5s` value is never used. Upstream #348 removes the
`120` override and keeps `connectionTimeout`. **Inherited by the merge.**

## 3. Findings — reconnection stability

### Finding 4 [Potential] — Gateway-registration ceiling `≈ 9s`

- `TelnyxRTC/Telnyx/TxClient.swift:130` — `DEFAULT_REGISTER_INTERVAL = 3.0`
- `TelnyxRTC/Telnyx/TxClient.swift:131` — `MAX_REGISTER_RETRY = 3`
- `updateGatewayState(newState:)` is defined at `:1069`; the retry/exhaustion logic is the `default` branch at `:1100-1122`.

Each non-`REGED` state reschedules `registerTimer` after `3s` (`:1109`); on expiry it
decrements `registerRetryCount` and either re-requests state or emits
`TxError.serverError(reason: .gatewayNotRegistered)` (`:1118`) when retries are exhausted.
Worst-case time-to-ready `≈ 3s × 3 = 9s`.

**Potential link:** contributes to Tom Clark's connection-delay reports; not isolated in
logs. **Recommendation:** document the worst-case; consider exposing the interval/retry
count (code change).

### Finding 2 [Potential] — `reconnectClient()` arms the `60s` reconnect timer before validating reconnect is possible

`TelnyxRTC/Telnyx/TxClient.swift:1612`:

```swift
func reconnectClient() {                                                            // :1612
    if self.isCallsActive {
        updateActiveCallsState(callState: CallState.RECONNECTING(reason: .networkSwitch)) // :1614
        startReconnectTimeout()                                                          // :1615
    } else { return }
    if let txConfig = self.txConfig {
        if (txConfig.reconnectClient) {
            guard let currentCall = self.calls[self.currentCallId] else {            // :1622
                Logger.log.e(message: "Current Call not available for ATTACH")
                return
            }
            currentCall.endForAttachCall()          // :1627
            self.socket?.disconnect(reconnect: true) // :1628
        } ...
    }
}
```

State transitions to `RECONNECTING` and the timer is armed (`:1614-1615`) **before** the
`guard` at `:1622`. If `currentCall` is missing, the function returns with the timer
already running — state stays `RECONNECTING` for `reconnectTimeout`
(`TxConfig.DEFAULT_TIMEOUT = 60.0`, `Models/TxConfig.swift:21`,`:67`), after which
`startReconnectTimeout()` fires `TxError.callFailed(reason: .reconnectFailed)`
(`TxClient.swift:1600`).

**Potential link:** see §0 (cold-start dead-air). **Recommendation:** validate the call
first, then transition state and arm the timer (code change). *Not* covered by the
upstream merge — verify against `4.0.2` reconnect code before fixing (it was not changed
between `3.2.0` and `4.0.2` in a way that resolves this).

### Finding 3 [Potential] — Reconnect thrash on network flapping — no debounce, no in-flight guard

`NetworkMonitor.shared.onNetworkStateChange` (`TelnyxRTC/Telnyx/TxClient.swift:329`) calls
`reconnectClient()` on every distinct transition:

```swift
case .wifi:            ...; self.reconnectClient()   // :336
case .cellular, .vpn:  ...; self.reconnectClient()   // :339
```

A Wi-Fi↔cellular handover fires these repeatedly. Each call runs
`currentCall.endForAttachCall()` + `socket?.disconnect(reconnect: true)` (`:1627-1628`)
with no coalescing and no `isReconnecting` flag — overlapping teardown/reattach cycles can
stack.

**Potential link:** see §0 (cold-start dead-air mechanism). **Recommendation:** debounce
transitions + add a concurrent-reconnect guard (code change).

### Finding 6 [Potential] — Fixed reconnect cadence, no backoff

- `TelnyxRTC/Telnyx/TxClient.swift:133` — `RECONNECT_BUFFER = 1.0`
- `TelnyxRTC/Telnyx/TxClient.swift:1724` — `DispatchQueue.main.asyncAfter(deadline: .now() + TxClient.RECONNECT_BUFFER)` in `onSocketDisconnected(reconnect:region:)`.

A flat `1.0s` delay precedes every reconnect attempt; under sustained loss the SDK retries
roughly every `~1s` for the `60s` window. **Recommendation:** exponential backoff with a
cap (code change).

### Finding 8 [Documentation] — `checkInternetAccess` reaches `https://www.google.com`

`TelnyxRTC/Telnyx/Utils/NetworkMonitor.swift:76` (`URL(string: "https://www.google.com")!`),
called only on VPN-classified paths (gated by `if newState == .vpn`, `NetworkMonitor.swift:42-43`).
Privacy/footprint note. **Recommendation:** use a Telnyx endpoint or path-flags-only
detection (code change).

## 4. ICE negotiation & audio-session timing

The original plan listed ICE/audio timing constants as a standalone section. **Per the
review, bare constants are not audit content; this section is retained only where a
constant bears on a stability finding above.**

- **Reconnect-adjacent audio resets** (relevant to Findings 2/3 and the cold-start dead-air hypothesis): on ICE `disconnected → connected` and `connected → disconnected` the SDK schedules an audio-device reset `+200ms` (`TelnyxRTC/Telnyx/WebRTC/Call.swift:1572`, `:1583`); ICE `failed` triggers an auto ICE-restart `+500ms` (`Call.swift:1545`). If Finding 3's thrash drives spurious disconnect→connect cycles, these resets fire repeatedly — a plausible audio-glitch contributor.
- **RTT-driven audio reset** (relevant to perceived audio quality under loss): high-RTT timer starts at `currentRttMs >= 500` (`Call.swift:1672`), a `5.0s` reset timer (`Call.swift:1694`), re-checked against `>= 1000` before resetting (`Call.swift:1705`).

The remaining constants from the plan — `NEGOTIATION_TIMOUT = 0.3` /
`TRICKLE_ICE_TIMEOUT = 3.0` (`Peer.swift:30`,`:34`), the `5.0s` ICE-restart gather wait
(`Peer+IceRestart.swift:60`), and the mute/unmute reset sequence (`Peer.swift:743`,`:753`)
— carry **no demonstrated link to a stability complaint** and are therefore **cut** from
the findings, recorded here only as verified-present (Appendix A) for future reference.
`CallTimingBenchmark` (`Utils/CallTimingBenchmark.swift`) should be enabled in the field
to convert these Potential links into Observed ones.

## 5. Upstream divergence (headline)

Verified against the real upstream repo (`team-telnyx/telnyx-webrtc-ios`): `origin/main`
`CHANGELOG.md` and `Services/Socket.swift`, plus the `4.0.1` tag.

**Fork state:** `feature/manual-audio-mode` = `3.2.3-fork` (SDK `3.2.0` + #337 cherry-pick
+ `manualAudioSessionManagement`). **Upstream `main` = `4.0.2`** (verified via
`origin/main:TelnyxRTC.podspec`), with #348 still under `[Unreleased]`.

> **Correction to the plan:** the plan stated upstream was at `4.0.1`; it has since moved
> to `4.0.2`. And #348's watchdog reuses the existing **`5s`** `connectionTimeout`, not a
> "`~10s` default" as the plan implied. Both corrected here.

### #348 verification (the headline fix)

Upstream `Socket.connect()` now (verified, `origin/main:TelnyxRTC/Telnyx/Services/Socket.swift`):

- calls `startConnectionTimeout()` **unconditionally** (no `shouldFallbackToAuto` guard);
- keeps `request.timeoutInterval = connectionTimeout` (no `120` override);
- sets `suppressDisconnectCallbackAfterConnectionTimeout = false` and suppresses the duplicate disconnect callback from a timed-out socket;
- redials on a stalled handshake, falling back to auto region only for genuine Telnyx hosts (`*.rtc.telnyx.com` / `*.rtcdev.telnyx.com`).

This **directly fixes Finding 1 and Finding 7**. Confirmed real bug, resolved upstream.

### Fork-vs-upstream gap table

| Item | Upstream version | In fork? | Maps to |
|------|------------------|----------|---------|
| Socket connect-timeout watchdog + redial ([#348](https://github.com/team-telnyx/telnyx-webrtc-ios/pull/348)) | `[Unreleased]` | ❌ | Findings 1, 7 |
| Push reattach respects `useTrickleIce` ([#343](https://github.com/team-telnyx/telnyx-webrtc-ios/pull/343)) | `4.0.1` | ❌ | Rewynd push/CallKit (A2/M-2) |
| Push-answer login race fix ([#345](https://github.com/team-telnyx/telnyx-webrtc-ios/pull/345)) | `4.0.1` | ❌ | Rewynd push/CallKit (A2/M-2) |
| Unsafe-UUID removal in push setup | `4.0.0` | ❌ | Rewynd push/CallKit (A2/M-2) |
| Missed-call notifications now opt-in (`enableMissedCallNotifications`) | `4.0.0` (breaking) | ❌ | Migration consideration |
| 401 push-socket ping fix ([#337](https://github.com/team-telnyx/telnyx-webrtc-ios/pull/337)) | `3.2.1` | ✅ (cherry-pick) | — |
| `manualAudioSessionManagement` patch | n/a (fork-only) | ✅ | Fork's reason for existing — **must preserve** |

### Merge-cost estimate: **MEDIUM**

`git diff 3.2.0 origin/main` touches **17** `.swift` files. The `manualAudioSessionManagement`
patch touches **3** (`TxClient.swift`, `Call.swift`, `Peer.swift`) + `TelnyxRTC.podspec`.
Overlap (both the patch **and** upstream changed) is **2 files**:

| File | Upstream churn (`3.2.0`→`4.0.2`) | Patch churn | Conflict risk |
|------|----------------------------------|-------------|---------------|
| `TelnyxRTC/Telnyx/TxClient.swift` | `+90 / −31` (`121` lines) | `+104` | **High** — both heavy, both in push/audio-enable paths; #348 + #345 rework the same connect/push code the patch hooks into. Expect hand-resolution. |
| `TelnyxRTC/Telnyx/WebRTC/Call.swift` | `+37 / −1` (`38` lines) | `+8` | Low–medium — small patch surface. |
| `TelnyxRTC/Telnyx/WebRTC/Peer.swift` | untouched upstream | `+9` | **None** — clean. |

Net: a real 3-way merge is needed in one file (`TxClient.swift`), trivial in the other
two. The patch is small (`+114` lines, localized), so re-applying it on top of `4.0.2` is
tractable. **Medium** cost. Note `4.0.0` is a **breaking** release (missed-call
notifications become opt-in via `enableMissedCallNotifications`) — review the v3→v4
migration guide before merging.

## 6. Test-coverage gaps (mapped to findings)

Each gap is justified by the finding it would have caught. Verified against
`TelnyxRTCTests/`.

| Missing test | Would have caught | Status today |
|--------------|-------------------|--------------|
| `NetworkMonitor` transition → reconnect behaviour | Finding 3 (thrash), Finding 2 (ordering) | 0 tests (`grep NetworkMonitor TelnyxRTCTests/` → none) |
| INVITE-timeout fires `ORIGINATOR_CANCEL`/`487` after `10s` | Finding 5 | 0 tests (`grep handleInviteTimeout/INVITE_TIMEOUT` → none) |
| Socket connect-timeout on `.auto` region | Finding 1, Finding 7 | 0 tests for `handleConnectionTimeout` on default host |
| Register-retry exhaustion → `gatewayNotRegistered` | Finding 4 | 0 tests |

Existing patterns to follow: `TelnyxRTCTests/TxClientReconnectTimeoutTests.swift` (timer
lifecycle), `TelnyxRTCTests/RegionIntegrationTests.swift` (`MockSocket`),
`TelnyxRTCTests/Helpers/RTCTestDelegate.swift` (delegate stubs).

## 7. Prioritized recommendations (split by deliverable category)

### A. Maintenance — upstream merge (highest leverage)

1. **Merge upstream `4.0.2` + #348; re-apply `manualAudioSessionManagement`.** Inherits the
   fix for Findings 1 & 7 and the push-path fixes (#343/#345/`4.0.0`) that map to the
   Rewynd push/CallKit items. Medium conflict cost (one file). Review the v3→v4 breaking
   change first.

### B. Code changes required (fork-specific; not resolved by the merge)

2. Reorder `reconnectClient()` to validate the current call before arming the `60s` timer
   (Finding 2). *Verify against `4.0.2` first.*
3. Debounce `NetworkMonitor` transitions + add an in-flight reconnect guard (Finding 3).
4. Exponential backoff on the reconnect cadence (Finding 6).
5. Add `TxConfig` knobs: `connectTimeout`, `inviteTimeout` (`reconnectTimeout` already
   exists, `TxConfig.swift:67`). Decide whether to adopt #348's region-fallback wholesale.
6. Replace the `google.com` reachability probe (Finding 8).
7. Add the missing tests in §6.

### C. Documentation-only (this deliverable)

8. This report. No further documentation work pending.

## Out of scope (explicit)

- No SDK/source, demo-app, podspec/Podfile, or verto-JSON edits — documentation only.
- App-side findings from the parent Rewynd audit (push/CallKit race, audio-session hack,
  SIP-credentials silent failure) are **app-repo concerns**, referenced here only where an
  upstream SDK fix would reduce app-side exposure. Not fixed here.

## Appendix A — Verification log

All citations verified against `feature/manual-audio-mode` (`3.2.3-fork`). The plan's
original citations used `TelnyxRTC/Classes/...` paths and line numbers from an unshipped
tree; corrections below. **No findings removed — all cited code is present.**

| Citation (this report) | Status | Note |
|------------------------|--------|------|
| `Services/Socket.swift:24` `connectionTimeout = 5.0` | ✓ verified | path corrected from plan's `WebRTC/Socket.swift` |
| `Services/Socket.swift:41` `request.timeoutInterval = connectionTimeout` | ✓ verified | |
| `Services/Socket.swift:47` `TimeInterval(120)` | ✓ verified | |
| `Services/Socket.swift:50` `shouldFallbackToAuto` guard | ✓ verified | |
| `Services/Socket.swift:86-93` `shouldFallbackToAuto` returns false on nil region | ✓ verified | |
| `Services/Socket.swift:95-102` `extractRegionPrefix` | ✓ verified | |
| `Models/TxServerConfiguration.swift:29` `region: Region = Region.auto` | ✓ verified | |
| `InternalConfig.swift:13` `prodHost = "wss://rtc.telnyx.com"` | ✓ added | plan asserted the host without a line; pinned here |
| `TxClient.swift:130` `DEFAULT_REGISTER_INTERVAL = 3.0` | ✓ verified | unchanged by patch |
| `TxClient.swift:131` `MAX_REGISTER_RETRY = 3` | ✓ verified | |
| `TxClient.swift:133` `RECONNECT_BUFFER = 1.0` | ✓ verified | |
| `TxClient.swift:195` `INVITE_TIMEOUT_SECONDS = 10.0` | ✓ corrected | was `:176` on `fork/main`; `+19` on shipped branch |
| `TxClient.swift:329-339` `onNetworkStateChange` → `reconnectClient()` (`:336`,`:339`) | ✓ corrected | was `:281-291` on `fork/main` |
| `TxClient.swift:810` `startInviteTimeout` timer | ✓ corrected | was `:762` |
| `TxClient.swift:823-846` `handleInviteTimeout` (`ORIGINATOR_CANCEL` `:828`, `487` `:829-830`) | ✓ corrected | was `:775-798` |
| `TxClient.swift:1069` `func updateGatewayState` | ✓ corrected | plan implied the func was at the retry-block range; header is here |
| `TxClient.swift:1100-1122` register retry/exhaustion; `gatewayNotRegistered` `:1118` | ✓ corrected | was `:1057-1077` |
| `TxClient.swift:1577` `func startReconnectTimeout`; `reconnectFailed` `:1600` | ✓ corrected | was `:1529`/`:1552` |
| `TxClient.swift:1612-1628` `reconnectClient` (`RECONNECTING` `:1614`, timer `:1615`, guard `:1622`, `endForAttachCall` `:1627`, `disconnect(reconnect:)` `:1628`) | ✓ corrected | was `:1564-1580` |
| `TxClient.swift:1724` `RECONNECT_BUFFER` usage | ✓ corrected | was `:1676` |
| `Models/TxConfig.swift:21` `DEFAULT_TIMEOUT = 60.0`; `:67` `reconnectTimeout` | ✓ verified | |
| `WebRTC/Call.swift:1545` ICE-restart `+0.5` | ✓ corrected | was `:1537` |
| `WebRTC/Call.swift:1572`,`:1583` audio reset `+0.2` | ✓ corrected | was `:1564`,`:1575` |
| `WebRTC/Call.swift:1672` `currentRttMs >= 500`; `:1694` `5.0` timer; `:1705` `>= 1000` | ✓ corrected | was `:1664`/`:1680`/`:1697`; the `>=1000` re-check was outside the plan's `1664-1686` range |
| `WebRTC/Peer.swift:30` `NEGOTIATION_TIMOUT = 0.3`; `:34` `TRICKLE_ICE_TIMEOUT = 3.0` | ✓ verified | unchanged by patch; **cut from findings** (no stability link) |
| `WebRTC/Peer.swift:743` mute step `+0.2`; `:753` `resetAVAudioSessionBuffers` | ✓ corrected/verified | `:738`→`:743`; `:753` unchanged; **cut from findings** |
| `WebRTC/Peer+IceRestart.swift:60` ICE-gather `5.0s` | ✓ verified | **cut from findings** |
| `Utils/NetworkMonitor.swift:76` `google.com`; `:42-43` `.vpn` gate | ✓ verified | |
| `Utils/CallTimingBenchmark.swift` milestones | ✓ verified | |

**Upstream §5 references** (verified against `origin/main` + `4.0.1` tag): #348
(`[Unreleased]`, Socket.swift behaviour confirmed line-by-line), #343 + #345 (`4.0.1`
changelog + tag), `4.0.0` breaking missed-call opt-in, #337 (`3.2.1`, present as
cherry-pick). Upstream version corrected `4.0.1` → `4.0.2`. #348 watchdog corrected
`~10s` → `5s`.
