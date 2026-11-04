# Unity — Experimental Cellular Call Audio Bridge

> **Design and feasibility README · Updated 2026-11-04**  
> **Target:** Samsung Galaxy M35 (Android 16 / One UI 8.5) ↔ Fedora Linux ↔ Bluetooth headphones  
> **Status:** Static Android permissions and audio routes **verified on the phone**; live call-audio extraction, injection, and the proposed Unity bridge **not yet verified**.

## Executive summary

The goal is to let a cellular call remain on the Android phone while **Unity transports the call's incoming and outgoing PCM audio over its existing authenticated LAN connection**. The Fedora laptop then sends incoming sound to any selected PipeWire output, including Bluetooth headphones, and captures outgoing speech from the selected laptop/headset microphone.

```text
INCOMING (downlink)
Cellular caller
  → carrier / phone modem
  → Android telephony RX → privileged AudioRecord
  → Unity Android → encrypted Unity LAN session
  → Linux unityd → PipeWire → laptop-connected Bluetooth headphones

OUTGOING (uplink)
Laptop microphone (or Bluetooth headset microphone)
  → PipeWire → Linux unityd
  → encrypted Unity LAN session → Unity Android
  → privileged AudioTrack → Android telephony TX
  → modem / cellular network → caller
```

This is **not** Bluetooth HFP between the phone and laptop. The headphones remain connected to the laptop, and Linux handles their Bluetooth audio profile. When a Bluetooth headset's microphone is used, Linux will typically switch to HFP/HSP rather than high-quality A2DP; using the laptop's own microphone can keep headphone playback on A2DP.

**Finding:** The Galaxy M35 has all four relevant permissions granted to `com.android.shell` and explicitly declares both telephony RX/TX devices and usable policy routes. This is strong **static feasibility evidence**, not proof of working call audio.

**Recommended approach:** Add an *optional, Advanced-tier, experimental* privileged Android audio backend behind Unity Phone's existing call controller; retain the existing HFP path as fallback. **Do not** replace Unity Phone, Unity Messages, the pairing system, or the existing media-audio feature.

---

## 1. Goals, non-goals and status vocabulary

### Goals

1. Answer, decline, dial, end, mute, and otherwise control calls using the **existing Unity Phone** interfaces.
2. During an explicitly authorized cellular call, deliver the remote person's audio to the laptop over LAN.
3. Deliver the laptop's chosen microphone input to the phone's cellular-call uplink over LAN.
4. Keep the Bluetooth headphones connected to **Fedora**, not to the Samsung phone.
5. Keep Unity Messages fully separate; call audio must never be stored in the messaging subsystem.
6. Reuse Unity's authenticated session, device identity, authorization, reconnect handling, and Linux sound-server integration.
7. Offer an understandable audio selector: **Phone**, **Computer (HFP)**, or **Computer (LAN bridge — experimental)**.
8. Stop safely on call end, authorization loss, helper death, transport loss, or user route change.

### Non-goals

- Bypassing a remote participant's consent or building a covert recorder.
- Recording calls, transcription, storing PCM, or relaying a conversation to third parties.
- Replacing the cellular modem, SIM, carrier, IMS, or Telecom call service.
- Routing emergency calls or service/USSD codes through experimental audio.
- Building a SIP/PBX server: Voice SIP Agent is a **technical reference**, not a dependency Unity needs.
- Modifying the system partition, rooting the phone, or depending on patched vendor mixer files.
- Replacing the default SMS role with Unity Phone or replacing the default Phone role with Unity Messages.
- Assuming `MODE_CALL_REDIRECT` is sufficient by itself or required for all devices.

### Status definitions

| Term | Meaning |
| --- | --- |
| **Verified on M35** | Direct evidence from ADB outputs supplied on 2026-11-04 |
| **Implemented in Unity** | Documented in the current Unity project repositories; platform acceptance limits may still apply |
| **Demonstrated externally** | Reported by an independent project, on its tested devices |
| **Proposed** | Design recommendation, **not** present in Unity yet |
| **Unknown** | Must be tested on this M35 and its specific firmware/carrier path |

## 2. Existing Unity architecture — what already works or exists

Unity is split into independent repositories. Respect their ownership boundaries.

| Component | Current responsibility | Relationship to proposed feature |
| --- | --- | --- |
| [Unity (hub)](https://github.com/MukhilDharan5/Unity) | Core / Basic / Advanced definitions and repository documentation | Classify call-audio interception as **Advanced**, optional |
| [Unity Android](https://github.com/MukhilDharan5/Unity_Android) | Authenticated companion, paired transport, local IPC bridges, Android integrations | Orchestrates authorization and PCM transport; does **not** become the dialer |
| [Unity Phone](https://github.com/MukhilDharan5/Unity_Phone) | Android default dialer/Telecom, contacts, call history; native Linux Phone client | **Sole authority** on call state, call ID, mute, and user audio choices |
| [Unity Messages](https://github.com/MukhilDharan5/Unity_Messages) | Independent Android default-SMS app and Linux SMS client | Remains independent; integrates through contacts, Call/Message handoffs |
| [Unity Linux](https://github.com/MukhilDharan5/Unity_Linux) | `unityd` Rust daemon, TLS transports, D-Bus, `unityctl`, GTK frontend, audio routing | Handles selected audio devices and proposed call-PCM session |

### Unity Phone — existing working architecture

- Unity Phone on Android is the default Phone app and owns `InCallService`, Telecom, calls, contacts and history. It can work without Unity or Linux.
- Unity Android relays **authorized AIDL commands and events** from Phone to paired desktops. It does not create a second dialer, call-controller service, or contacts database.
- Unity Phone on Linux (GTK 4/libadwaita) is the call UI: incoming/outgoing/call-waiting windows, call controls, contacts, history, keypad, SIM choice and audio selection.
- The main Unity application deliberately does **not** have a duplicate call UI.
- The current optional **computer call audio** route uses Bluetooth hands-free/HFP and Linux's sound-server integration. This existing feature is the **fallback**, not something the LAN method should overwrite.
- Unity Phone's *audio selection version* and cancellation logic protect explicit phone-side route choices from stale computer-audio completions. The new backend must honor the same ordering.

**Important:** Unity Phone's current call-state relay is *control and metadata*, **not raw cellular audio**. Adding a PCM bridge is new functionality.

### Unity Messages — existing working architecture

- Unity Messages is a separate Android native SMS application with a GTK/libadwaita Linux client.
- Android owns SMS, SIM selection and carrier delivery reports; the Linux UI is a thin client over Unity's existing authenticated transport.
- It has its own default-SMS consent/role; Unity Phone retains the default-Phone role.
- Existing integration: Messages can open Unity Phone for calls; a contact's **Message** action can open Unity Messages. Independent conversations, drafts, read state, archive/pin controls and notifications remain in the messaging app.
- Current stated scope is **SMS only** (no MMS/RCS/iMessage); this audio feature does not change that.

**Proposed Messages integration:** No new call-audio privilege or message database access. Optionally expose a passive, non-sensitive “In a cellular call on this device” presence indicator where helpful. Tapping **Call** continues to open Unity Phone. Message delivery remains unaffected when the LAN audio bridge fails.

### Unity media audio — existing, but not cellular-call capture

- Unity can carry *laptop media → phone* audio and *permitted phone media → laptop* audio over selected LAN/Bluetooth transports.
- The phone-to-laptop media direction uses Android **AudioPlaybackCaptureConfiguration + user-approved MediaProjection**, which excludes cellular call audio and protected audio.
- The current documented LAN media format is 48 kHz stereo 16-bit PCM in 10 ms packets; the Classic Bluetooth path is 16 kHz mono PCM in 10 ms packets.
- Transport already uses pinned TLS 1.3, one-use stream tokens, bounded framing and route restoration. Reuse these principles, **not** the media capture API itself.
- The main Unity Media page and Unity Phone's **call audio** route selector must remain distinct.

Project references: [Linux telephony guide](https://github.com/MukhilDharan5/Unity_Linux/blob/main/docs/telephony.md), [Android Phone relay](https://github.com/MukhilDharan5/Unity_Android/blob/main/docs/phone-relay.md), [Media routing](https://github.com/MukhilDharan5/Unity_Linux/blob/main/docs/audio-routing.md), [Phone Linux README](https://github.com/MukhilDharan5/Unity_Phone/blob/main/linux/README.md), [Messages README](https://github.com/MukhilDharan5/Unity_Messages/blob/main/README.md).

---

## 3. Android technical basis

### 3.1 Why ordinary application-level capture fails

Cellular voice commonly travels through modem/telephony interfaces and a vendor Audio HAL/DSP path, not an ordinary app-playback mix. MediaProjection-based recording only captures permitted media streams; it does **not** grant access to raw cellular voice. Normal `AudioRecord` microphone capture is not equivalent to voice downlink extraction.

Android instead exposes privileged telephony input/output devices and internal call-audio APIs. Availability depends on the framework **and** actual vendor audio policy/HAL.

### 3.2 Relevant privileged APIs

In AOSP's `AudioManager` system API:

```java
boolean isPstnCallAudioInterceptable();

AudioRecord getCallDownlinkExtractionAudioRecord(AudioFormat format);

AudioTrack getCallUplinkInjectionAudioTrack(AudioFormat format);
```

- The first reports the framework's advertised interceptability capability, based on the telephony RX/TX devices being available.
- Downlink extraction is for the far-end voice; uplink injection is for audio supplied to the caller.
- These APIs are **system/hidden**, not an ordinary third-party SDK contract.
- `MODE_CALL_REDIRECT` (integer **5**, API 33+) can modify cellular-call audio routing; it does **not** single-handedly open two PCM endpoints.
- Call PCM APIs may be usable while the active call is still in `MODE_IN_CALL`; forcing `MODE_CALL_REDIRECT` can break routing on some vendors.
- A routing-mode switch, if ever used, requires explicit ownership and restoration; it may interfere with earpiece/microphone paths.
- AOSP main-branch behavior is a starting point; **the Samsung firmware on the phone is authoritative** for actual behavior.

### 3.3 Required privileged permissions

`com.android.shell` may hold:

```text
android.permission.CALL_AUDIO_INTERCEPTION
android.permission.CAPTURE_AUDIO_OUTPUT
android.permission.MODIFY_AUDIO_ROUTING
android.permission.MODIFY_PHONE_STATE
```

A regular APK does **not** inherit these. The privileged part must run with the appropriate **shell identity (UID 2000)**, e.g. a separately scoped Shizuku UserService or ADB-launched `app_process` probe.

Shell UID is not a magic universal bypass: Android may still check **package attribution**, SELinux, active call state, resource ownership, stream attributes and vendor routing. A privileged helper can be granted the permissions yet still receive only silence.

Do **not** expose a generic “execute arbitrary privileged command” RPC to Unity's paired computers. Grant only tightly scoped call-audio operations to a verified caller, and bind them to an actual active call.

### 3.4 Key external precedent

[Voice SIP Agent](https://github.com/sparkcyf/android-voice-sip-agent) demonstrates a no-root cellular↔SIP gateway using Shizuku/shell services and telephony audio routing. Its [technical write-up](https://blog.sparktour.me/en/posts/2026/10/10/android-shizuku-cellular-sip-gateway/) explains:

- shell-context and **Binder attribution** problems;
- exact capture/injection audio attributes (not ordinary `AudioRecord` defaults);
- direct routing versus call redirection, local muting and echo problems;
- clocking and jitter buffers;
- call control, call lifetime and safe cleanup;
- vendor-specific failures and the requirement for real-device listening tests.

**Do not infer M35 compatibility from other devices.** The external reports show *feasibility of the technique*, not proof on this Samsung build. Also check the project's per-file licensing before using code.

---

## 4. Galaxy M35 — actual phone checks (2026-11-04)

**Device under test:** Samsung Galaxy M35, Android 16 / One UI 8.5; unrooted phone, using ADB shell. The exact build fingerprint, carrier and Wi-Fi-calling status were **not** recorded in the supplied output.

### 4.1 Shell UID and SELinux context: **PASS**

Command:

```sh
adb shell id
```

User-supplied output (excerpt):

```text
uid=2000(shell) gid=2000(shell) ... context=u:r:shell:s0
```

**Conclusion:** Commands executed under expected ADB `shell` UID and context. This does **not** prove a future Shizuku service's *effective attribution*.

### 4.2 Permissions: **PASS**

Command:

```sh
adb shell dumpsys package com.android.shell |
  grep -E 'CALL_AUDIO_INTERCEPTION|CAPTURE_AUDIO_OUTPUT|MODIFY_AUDIO_ROUTING|MODIFY_PHONE_STATE'
```

The output explicitly included **`granted=true`** for:

| Permission | On-device status |
| --- | --- |
| `CALL_AUDIO_INTERCEPTION` | `granted=true` |
| `CAPTURE_AUDIO_OUTPUT` | `granted=true` |
| `MODIFY_AUDIO_ROUTING` | `granted=true` |
| `MODIFY_PHONE_STATE` | `granted=true` |

Repeated granted lines are not evidence of errors; the dump has several permission sections.

### 4.3 Telephony RX/TX devices: **PASS (declared by audio policy)**

Command:

```sh
adb shell dumpsys media.audio_policy |
  grep -iE -C 5 'telephony|incall|in.call|voice_tx|voice_rx'
```

Actual M35 evidence:

```text
Port ID: 11; "Telephony Tx"; {AUDIO_DEVICE_OUT_TELEPHONY_TX, @:}
Port ID: 15; "Telephony Rx"; {AUDIO_DEVICE_IN_TELEPHONY_RX, @:}
```

**Interpretation:** The necessary telephony input/output device types exist in Samsung's audio policy.

### 4.4 Declared capture and injection routes: **PASS (static)**

Key excerpts:

```text
Mix; Sink: "Telephony Tx"
  Sources: "incall_music_uplink"

Mix; Sink: "primary input"
  Sources: ... "Telephony Rx" ...
```

Further output:

```text
IOProfile name:incall_music_uplink
Latency: 40
AUDIO_FORMAT_PCM_16_BIT; 48000; Channel mask: 0x3
Devices: {AUDIO_DEVICE_OUT_TELEPHONY_TX, @:}
Global active count: 0
```

And the existing policy patch:

```text
Audio Patches (2):
  owner uid 1041; ...
  [src 1] Mix Port ID: 10; I/O handle: 45
  [sink 1] Device Port ID: 11; {AUDIO_DEVICE_OUT_TELEPHONY_TX, @:}
```

Other observed state:

```text
-STRATEGY_CALL_ASSISTANT
  Selected Device: {AUDIO_DEVICE_OUT_TELEPHONY_TX, @:}

mMode=0
isInWifiCall=0 (0x0)
```

**Interpretation and cautions:**

- `incall_music_uplink` exposes a telephony-directed output profile, here showing PCM 16-bit at 48 kHz; its profile also lists 8 kHz and 16 kHz among supported rates.
- The `primary input` has `Telephony Rx` as a declared source.
- The **40** is a reported profile latency in ms; it is **not measured end-to-end conversational latency**.
- `curOpenCount: 1` and `Global active count: 0` mean an output is open but not actively producing a call stream at snapshot time.
- `mMode=0` corresponds to normal mode, so these were **not captured during a confirmed active cellular call**.
- An existing policy patch does **not** prove that synthetic uplink PCM can be heard by the remote caller.

### 4.5 Compatibility checkpoint matrix

| Check | Result | Confidence |
| --- | --- | --- |
| ADB shell UID 2000 | **PASS** | Measured |
| Four privileged permissions granted | **PASS** | Measured |
| `AUDIO_DEVICE_IN_TELEPHONY_RX` exists | **PASS** | Policy dump |
| `AUDIO_DEVICE_OUT_TELEPHONY_TX` exists | **PASS** | Policy dump |
| Static route: Telephony Rx → primary input | **PASS** | Policy dump |
| Static route: `incall_music_uplink` → Telephony Tx | **PASS** | Policy dump |
| Framework `isPstnCallAudioInterceptable()` returns true | **NOT TESTED** | Needs compiled shell probe |
| Shizuku helper initializes both streams | **NOT TESTED** | Needs actual helper |
| Non-silent far-end downlink during a call | **NOT TESTED** | Needs consenting live call |
| Remote party hears injected PCM | **NOT TESTED** | Needs consenting live call |
| Phone mic/earpiece suppression without losing bridge | **NOT TESTED** | Vendor-specific |
| Two-way Unity LAN + Fedora Bluetooth playback | **NOT IMPLEMENTED** | Proposed |
| Call-end/disconnect automatic restoration | **NOT IMPLEMENTED** | Proposed |

**Bottom line:** The M35 passes static permission and route prerequisites. Its actual **runtime interception capability is still unknown**.

---

## 5. Problems, causes, possible solutions and present status

| Problem | Why it happens | Recommended handling | Status |
| --- | --- | --- | --- |
| Ordinary phone audio capture produces silence / mic sound | Cellular audio isn't ordinary app playback; privileged source | Shell-context extraction `AudioRecord`, only on compatible HAL | **Proposed; static prerequisites passed** |
| Uplink stream plays locally but caller hears nothing | Wrong route, attributes, mode, HAL or call state | Exact telephony injection API, verify on second phone | **Unverified** |
| Shell permissions granted but API throws or silently fails | Attribution UID/package mismatch; hidden API checks | Scoped Shizuku UserService with correct shell context; log exact error | **Known external issue** |
| `MODE_CALL_REDIRECT` breaks calls or mutes both directions | Vendor policy differs | Test **direct** call audio first; make redirect an optional strategy | **Proposed** |
| Phone microphone leaks into injected uplink | Physical mic may mix with injection | Unity Phone-authorized mute; verify injection stays audible | **Unverified on M35** |
| Phone earpiece continues to play remote audio | Physical route persists | Prefer a supported route change; optional redirect only after live validation | **Unverified** |
| Echo/feedback | Phone mic, laptop speakers or parallel routes | Prefer headset; prevent two simultaneous sinks/mics; echo controls | **Proposed** |
| LAN packet jitter / Wi-Fi powersaving | Uneven packet arrivals | Bounded adaptive jitter buffer; sequence numbers; deadline/underrun counters | **Can reuse Unity transport primitives** |
| Audio clock drift | Telephony and PipeWire/BT device clocks differ | Adaptive resampling / drift compensation and bounded queues | **Proposed** |
| Bluetooth headset mic degrades playback fidelity | HFP/HSP may replace A2DP when mic active | Offer laptop mic while headphone output remains A2DP | **OS/profile behavior to test** |
| Wi-Fi disappears mid-call | LAN audio media lost; call still exists on phone | Stop injection, restore a safe phone audio route; never hang up automatically | **Proposed** |
| Shizuku unavailable after reboot | Non-root Shizuku needs starting again | Disable LAN call option and preserve Phone/HFP fallback; clear setup instructions | **Known limitation** |
| Android/One UI update breaks hidden APIs | Not public SDK; vendor changes | Capability probe/version matrix; fail closed and keep fallback | **Ongoing risk** |
| Routing selection race | User changes phone audio while session is establishing | Extend existing audio selection version/cancel semantics | **Mechanism exists; new integration proposed** |
| Phone + Messages ownership conflict | Multiple apps could try to own Telecom/SMS | Keep Phone as default dialer; Messages as default SMS; Unity only transports | **Existing architecture** |
| Sensitive audio crosses devices | Raw speech exposes a new privacy boundary | Explicit opt-in, trusted paired devices, encrypted transport, no recording | **Required design** |
| Generic power/battery overhead | Real-time audio, radios, wakelocks | Active-call-only processing, bounded work, suspend cleanly | **Proposed** |
| Carrier-specific VoLTE/VoWiFi implementation | HAL may not expose that call's actual media | Test each SIM and call technology; fall back for unsupported cases | **Unverified** |

### Alternatives and fallback options

| Route | Two-way phone call on laptop? | Strength | Limitation |
| --- | --- | --- | --- |
| **Existing Unity Bluetooth HFP computer-audio path** | Yes, when supported and validated | Standard telecom headset behavior | BT/profile integration, connection/codec issues |
| **Proposed shell PCM → Unity LAN → PipeWire** | Potentially yes | Headphones stay on laptop; flexible output/mic choice | Hidden APIs, Shizuku, HAL and runtime checks |
| **Bluetooth multipoint** | Call primarily via phone-connected headset | Little software development | Does not implement the same laptop audio pipeline |
| **Unity existing media-audio forwarding** | **No cellular-call audio** | Existing framed/encrypted audio stack | Android playback-capture restrictions |
| **AudioRelay/SoundWire-like media transfer** | Not a complete cellular uplink/downlink solution | Simple for media | No privileged PSTN source/sink |
| **Root/system-privileged install** | Potentially | Broader access and system integration | Explicitly outside rootless goal |

---

## 6. Proposed Unity architecture

### 6.1 Ownership boundary

```text
                           ANDROID
    ┌───────────────────────────────────────────────────────┐
    │ Unity Phone (existing app)                            │
    │  - sole Telecom / default dialer / call state owner   │
    │  - validates call ID, mute, route, user intent         │
    │  - optional LAN audio setting in call screen          │
    └───────────────────┬───────────────────────────────────┘
                        │ explicitly authorized local AIDL
    ┌───────────────────▼───────────────────────────────────┐
    │ Unity Android companion (existing)                    │
    │  - signed local IPC bridge and pairing permissions    │
    │  - session coordinator for LAN call audio             │
    │  - forwards state/events; does not take Telecom role  │
    └────────┬───────────────────────────────────┬──────────┘
             │ authenticated narrow IPC          │ pinned TLS 1.3
    ┌────────▼─────────────────────┐             │
    │ Proposed Call PCM helper     │             │
    │ Shizuku shell UID 2000       │             │
    │  - capability check          │             │
    │  - downlink AudioRecord      │             │
    │  - uplink AudioTrack         │             │
    │  - route/mode cleanup        │             │
    └──────────────────────────────┘             │
                                                ▼
                                        LINUX (existing)
                                  ┌────────────────────────────┐
                                  │ unityd + authenticated TLS │
                                  │ call PCM service (new)     │
                                  │ PipeWire adapter (new)     │
                                  └──────────────┬─────────────┘
                                                 │
                                     ┌───────────▼───────────┐
                                     │ Selected PipeWire      │
                                     │ output / microphone     │
                                     │ e.g. BT headphones     │
                                     └───────────────────────┘

    Unity Messages (existing, independent):
      default SMS app ↔ Unity Android relay ↔ Unity Messages Linux
      no PCM access, no Telecom ownership, no new audio privileges
```

### 6.2 Recommended module split (names are **proposed**, not current source paths)

| Layer | Proposed unit | Responsibility |
| --- | --- | --- |
| Unity Phone Android | `CallAudioRouteController` | Explicit route choice, active-call authorization, versioning and UI |
| Unity Android | `CallAudioBridgeCoordinator` | Binds Phone state to paired-session + helper lifetime |
| Android privileged helper | `CallPcmProbe` and `CallPcmSession` | Capability checking, audio open/read/write, limited route control |
| Unity protocol | `call_audio` capability/messages | Start/ready/state/stop/errors; scoped one-use session token |
| `unityd` | `call_audio` feature module | Session management, backpressure, metrics, shutdown |
| Linux audio adapter | `PipeWireCallAudio` | Capture selectable microphone, play to selected output, restore routes |
| Unity Phone Linux | `ComputerAudioRoutePicker` | Shows availability, selected mic/headphones, warning and failure recovery |
| Unity Messages | **No new audio module** | Existing call/message navigation only |

The module names above are conceptual. Integrate with the current `unityd` crates and versioned D-Bus contracts rather than claiming they are already implemented.

### 6.3 Call control and audio are separate planes

**Control plane (already exists):**

`Unity Phone Android → local signed AIDL → Unity Android → authenticated Unity transport → unityd → D-Bus → Unity Phone Linux`

It carries call identity, state, user intent, route selection, call actions and failure status. Remote emergency dialing/service codes remain denied.

**Audio plane (new):**

`Privileged Android PCM helper ↔ Unity Android ↔ dedicated authorized TLS media stream ↔ unityd ↔ PipeWire`

It carries only short PCM frames and operational statistics. No audio payload goes through the contact/SMS APIs, a contact database, notification log, or normal JSON metadata event queue.

### 6.4 UI behavior in Unity Phone

Suggested call audio section:

```text
Call audio
  ○ Phone
  ○ Computer — Bluetooth hands-free (existing)
  ○ Computer — Local network (experimental)

Computer audio devices
  Output:     [Bluetooth headphones ▼]
  Microphone: [Laptop microphone ▼]

Status: Connected · LAN · call audio active
        [Stop computer audio]
```

- Present LAN option **only** when phone capability, Shizuku access, authorized pairing, and media transport are available.
- **Never** change routes just because the UI is opened or refreshed.
- Confirm the call selected by a stable current call ID, user-selected target computer, and a fresh route selection version.
- A later user choice (Phone or HFP) **wins** over an in-flight LAN setup.
- On failure, explain whether it was permission, HAL, call state, no Wi-Fi, PCM silence, device output, or helper crash.

### 6.5 Messages coexistence

- Calls started from a Messages contact open **Unity Phone** just as today.
- Incoming calls while using Messages appear via Unity Phone's existing background call window / notification.
- The Messages Linux UI and Android SMS receiver are not involved in audio extraction, transport, or PCM processing.
- Preserve SMS send/delivery behavior even if Unity call PCM has an exception or reconnect.
- Do not make “Messages installed” a requirement for the call-audio capability.

### 6.6 Proposed network/audio protocol (illustrative, **not implemented**)

Reuse Unity's established pinned TLS 1.3, device grants, one-time media token and transport generation. Introduce an audio-specific capability, not a generic shell execution endpoint.

**Illustrative control message:**

```json
{
  "feature": "call_audio",
  "action": "start",
  "call_id": "<current-authorized-call-id>",
  "selection_version": 42,
  "source": "pstn_downlink",
  "sink": "pstn_uplink",
  "preferred_transport": "lan",
  "format": {
    "encoding": "s16le",
    "sample_rate": 16000,
    "channels": 1,
    "frame_ms": 20
  }
}
```

This is a **design sketch**, not an existing Unity API or final schema. Negotiate the real supported format rather than hard-coding 16 kHz.

Suggested audio packet metadata: version, generation/session ID, sequence, sample count, and compact timestamps. Carry PCM bytes as binary frames. Reject stale session generations, frames exceeding negotiated size, replayed authorization and wrong transport.

**Format budget examples:** 16 kHz × 16-bit × 1 channel = **32,000 bytes/s per direction**, so 20 ms = **640 bytes per PCM frame**, excluding headers and encryption. The device's observed `incall_music_uplink` output profile was active/open at a **48 kHz** configuration in the policy dump; the actual best call-tap format must be tested.

**Timing design:**

- Prefer 10–20 ms telephony frame units.
- Start with a small **bounded** jitter buffer (not an absolute 20 ms underrun rule).
- Instrument scheduling delay, decoder/read underruns, buffer occupancy, drops and audio drift.
- Do not independently pace the two directions forever without handling sample-clock drift.
- Prefer bounded latency over unbounded buffering; provide adaptive sample-rate correction or explicit drift compensation.
- Distinguish routing latency, LAN transport delay, PipeWire, Bluetooth codec and hardware delay in measurements.
- Avoid assuming `AudioFlinger` or the baseband always supports a particular rate or HAL path.

### 6.7 Session state machine (proposed)

```text
UNAVAILABLE
   └─ capability + consent + active call + LAN ready ─► AVAILABLE
AVAILABLE
   └─ explicit user selection ─► PREPARING
PREPARING
   ├─ verified audio streams + laptop sink/source ─► ACTIVE
   ├─ newer route selection ─► CANCELLED / restore
   └─ timeout / failed route ─► FAILED / restore
ACTIVE
   ├─ user chooses Phone/HFP ─► STOPPING / restore
   ├─ call ends ─► STOPPING / restore
   ├─ network loss ─► STOPPING / restore
   ├─ Phone authorization revoked ─► STOPPING / restore
   └─ helper death ─► STOPPING / restore
STOPPING
   └─ owned resources released ─► AVAILABLE or UNAVAILABLE
```

The **cellular call itself must not be ended** as the default reaction to media bridge failure. Release bridge ownership and restore a safe phone route instead; only the existing Phone call controller may decide on hang-up through an explicit permitted call action.

### 6.8 Audio ownership and local muting

Test separate questions on a real call:

1. Does downlink capture work without `MODE_CALL_REDIRECT`?
2. Does uplink injection reach the remote phone while the ordinary phone microphone is live?
3. Does using Unity Phone's **mute** suppress the physical handset microphone while leaving injected uplink alive?
4. Does `MODE_CALL_REDIRECT` silence local output **without** silencing the helper on M35?
5. Is it possible to keep normal phone fallback ready without persistent route damage?

Do not assume Android's `setMuted(true)` and voice injection are independent on Samsung merely because they were independent on another vendor's hardware. Prefer a **direct-path-first** strategy, with a mode-changing alternative only if testing proves it safe.

---

## 7. Implementation milestones and acceptance gates

### M0 — Read-only capability probe

- [x] Confirm shell UID/context.
- [x] Confirm shell permission grants.
- [x] Confirm telephony RX/TX policy devices.
- [x] Confirm declared input and `incall_music_uplink` output routes.
- [ ] Record `getprop ro.build.fingerprint` and relevant Android/One UI versions.
- [ ] Run compiled shell-context `AudioManager.isPstnCallAudioInterceptable()` probe.
- [ ] Capture diagnostic exceptions and permission/attribution errors only; do **not** record audio.

**Pass:** helper can run and framework reports support.  
**Fail:** capability false, permission failure or unsupported API; keep HFP fallback.

### M1 — Downlink-only live call

- [ ] Use a consenting caller on a second phone and a non-emergency test number.
- [ ] With active call, open correct `AudioRecord` using phone-specific framework attributes.
- [ ] Verify actual intelligible far-end speech; don't mistake random nonzero values or room mic sound for call capture.
- [ ] Test cellular/VoLTE versus Wi-Fi calling as separate cases where available.
- [ ] Verify stop/release after hang-up.

**Pass:** far-end voice recovered reliably, with no unexpected content.

### M2 — Uplink-only live call

- [ ] Create the privileged `AudioTrack` and route audio toward telephony TX.
- [ ] Send a safe, low-volume identifiable test tone / spoken sample.
- [ ] Have the remote participant confirm reception.
- [ ] Repeat with and without phone-side mute (where permitted).
- [ ] Verify injection disappears immediately on stop.

**Pass:** remote caller hears injected signal without losing the call or trapping the phone's normal route.

### M3 — Two-way Android-local proof

- [ ] Open both directions simultaneously.
- [ ] Try direct route first; optional redirect mode second only if necessary.
- [ ] Test local-mic leakage, earpiece audibility and echo.
- [ ] Test call waiting, incoming/outgoing, and selected SIM as separate edge cases.
- [ ] Validate restoration after helper crash and caller hang-up.

**Pass:** clear two-way call for a sustained test, no route ownership leak.

### M4 — Unity LAN transport

- [ ] Add versioned capability, authorization, session-generation and stop semantics.
- [ ] Reuse existing encrypted media stream and one-use token design.
- [ ] Add binary PCM framing, bounded buffering, sample-clock management and metrics.
- [ ] Test one-way loss, network switch, reordering/stale messages and reconnect.
- [ ] Ensure a LAN loss **does not terminate the phone call**.

**Pass:** end-to-end duplex PCM over the paired LAN session with bounded latency and safe reconnect failure.

### M5 — Unity Phone / PipeWire UI

- [ ] Add LAN route behind an explicit **experimental** toggle.
- [ ] Add independent output and microphone selectors.
- [ ] Preserve explicit Phone/HFP route choices and selection version.
- [ ] Make the Unity Phone desktop call window the only call-audio control UI.
- [ ] Test BT headphone output with laptop mic and BT headset mic separately.
- [ ] Keep Unity Messages completely unaffected.

**Pass:** a user can answer in Unity Phone on Linux and converse through the selected laptop devices.

### M6 — Stability and release guardrails

- [ ] Run 15–30 minute calls and measure one-way latency and drift.
- [ ] Repeat on at least two network conditions and with locked screen/Doze.
- [ ] Test Shizuku death, helper crash, Bluetooth disconnect and route switching.
- [ ] Validate emergency/service-code exclusion.
- [ ] Validate consent revocation and sensitive-data cleanup.
- [ ] Freeze a compatibility matrix per Android build, carrier and routing strategy.
- [ ] Require explicit opt-in each time the feature is first enabled for a trusted device.

**Pass:** predictable failure/recovery and repeatable real-device acceptance. Do not call it production-ready on static evidence.

---

## 8. Reproducible diagnostic commands (safe/read-only)

These do not start or record a phone call.

```sh
# 1. Shell identity
adb shell id

# 2. Requested/granted shell permissions
adb shell dumpsys package com.android.shell |
  grep -E 'CALL_AUDIO_INTERCEPTION|CAPTURE_AUDIO_OUTPUT|MODIFY_AUDIO_ROUTING|MODIFY_PHONE_STATE'

# 3. Telephony policy devices and relevant routing snippets
adb shell dumpsys media.audio_policy |
  grep -iE -C 5 'telephony|incall|in.call|voice_tx|voice_rx'

# 4. Optional Android build identity for a compatibility log
adb shell getprop ro.build.fingerprint
adb shell getprop ro.build.version.release
adb shell getprop ro.build.version.sdk

# 5. Optionally store the complete audio policy locally for comparison
adb shell dumpsys media.audio_policy > audio-policy-m35.txt
```

**Caution:** A full policy dump may contain Bluetooth MAC addresses and other device/session metadata. Remove private identifiers before publishing. No `adb shell` one-liner on its own verifies a live telephony PCM bridge; the next definitive check requires a compiled helper running in the correct shell context.

**Suggested test record template:**

```text
Device: Galaxy M35
Firmware fingerprint: <record>
Android / One UI: 16 / 8.5 (verify on-device)
Call type: carrier cellular / VoLTE / VoWiFi
SIM/carrier: <record privately>
Method: direct / redirect
Capability method return: true / false / exception
Downlink voice: yes / silence / mic-only / unknown
Uplink injection: audible / silent / unstable / unknown
Phone mic leakage: yes / no / unknown
Phone earpiece: audible / silent / unknown
Route restored after stop: yes / no / unknown
Notes: <redacted logs>
```

---

## 9. Security, privacy and release safeguards

This is an **Advanced** capability because it exposes live communication audio, not just call metadata.

- Require an **explicit, revocable authorization** for call PCM routing to a particular paired laptop. Pairing alone does not grant it.
- Do not route a call in the background just because the laptop is connected; selection must be user-driven or an explicitly configured, narrowly scoped preference.
- Display an unambiguous active-call audio indicator on **Android and Linux**.
- Bind media start to an **existing, currently active, authorized call**, trusted device ID and current call-state generation.
- Keep the shell helper's IPC API minimal: capability query, start, stop, health/status, and bounded PCM exchange.
- Permit only authenticated/signed local callers of the helper; never turn Shizuku into a public privileged daemon.
- Honor OS role changes, Phone consent revocation, disconnect and user stops immediately.
- Avoid disk buffers, recordings, content-aware logging and unencrypted audio debug dumps by default.
- Do not replay previous call sessions on network recovery or device restart.
- Treat BT, LAN and future transport sessions as different security contexts; reject token reuse.
- Provide a visible, tested fallback: phone's own mic/earpiece or already-supported HFP.
- Do not change emergency-call audio behavior, remote emergency dialing rules, or OS call ownership.
- Check local rules and all participants' consent before conducting live audio tests or implementing any future recording feature.

## 10. Open technical questions

1. Does the actual M35 framework report `isPstnCallAudioInterceptable() == true`?
2. Which exact format and attributes work with **Samsung's** `AudioRecord`/`AudioTrack` implementation?
3. Does the device expose voice PCM for **both VoLTE and VoWiFi** calls?
4. Is injecting into telephony TX mixed with the phone's microphone? Can that mic be muted without blocking injection?
5. Is `MODE_CALL_REDIRECT` usable, unnecessary, or harmful on M35?
6. Which process context and attribution package does this Samsung firmware require?
7. What happens if Bluetooth headphones connect/disconnect while PipeWire playback is active?
8. What is the actual measured one-way latency, and does it remain bounded over a long call?
9. Does a brief LAN interruption allow a clean fallback to local Phone audio without disconnecting the call?
10. How should capability availability be represented in Unity Phone's existing versioned audio selection protocol?

---

## 11. References and source-of-truth hierarchy

**Priority:** on-device behavior and reproducible tests → Android/AOSP implementation and system documentation → current Unity source → external implementations and reports. Neither a successful public demo nor a static policy listing guarantees M35 runtime behavior.

### Current Unity repositories

1. [Unity hub / project boundaries](https://github.com/MukhilDharan5/Unity)
2. [Unity Linux — telephony and messages documentation](https://github.com/MukhilDharan5/Unity_Linux/blob/main/docs/telephony.md)
3. [Unity Linux — existing audio routes and transport format](https://github.com/MukhilDharan5/Unity_Linux/blob/main/docs/audio-routing.md)
4. [Unity Android — dedicated Phone relay / call ownership](https://github.com/MukhilDharan5/Unity_Android/blob/main/docs/phone-relay.md)
5. [Unity Phone — Android and Linux integration](https://github.com/MukhilDharan5/Unity_Phone)
6. [Unity Phone — Linux UI](https://github.com/MukhilDharan5/Unity_Phone/blob/main/linux/README.md)
7. [Unity Messages — independent Android/Linux SMS](https://github.com/MukhilDharan5/Unity_Messages)

### Android and external research

8. [AOSP system API surface — `AudioManager` call downlink/uplink methods](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/api/system-current.txt)
9. [Android Developers — `AudioManager` / `MODE_CALL_REDIRECT`](https://developer.android.com/reference/android/media/AudioManager#MODE_CALL_REDIRECT)
10. [Android Developers — `InCallService`](https://developer.android.com/reference/android/telecom/InCallService)
11. [AOSP — audio architecture](https://source.android.com/docs/core/audio)
12. [AOSP — configurable audio policy, device/mix ports and routes](https://source.android.com/docs/core/audio/implement-policy)
13. [Android Developers — media playback capture restrictions](https://developer.android.com/media/platform/av-capture)
14. [Shizuku — setup and reboot constraints](https://shizuku.rikka.app/guide/setup/)
15. [Voice SIP Agent — source and current limitations](https://github.com/sparkcyf/android-voice-sip-agent)
16. [Voice SIP Agent — technical design, attribution/audio attribute pitfalls, clocking](https://blog.sparktour.me/en/posts/2026/10/10/android-shizuku-cellular-sip-gateway/)

> **Licensing note:** Use external projects as design references unless license compatibility and the exact licensing of copied files are verified. This README itself does not import third-party code.

---

## Current decision

**Proceed to a minimal read-only shell capability helper, then a consent-based live downlink/uplink probe.** Do not yet wire an experimental stream into Unity Phone's production call routing. Static M35 checks are encouraging, but only a real call can establish working extraction, injection, local muting and reliable restoration. Once those pass, reuse Unity's existing encrypted network/audio infrastructure and keep Unity Phone as call owner, Unity Messages as SMS owner, and Bluetooth HFP as fallback.
