# AppMic

AppMic is a Windows x64 VST2 plugin for **Equalizer APO 1.4.2 x64** that captures the playback of a selected application and mixes it into the microphone path while the application continues playing normally through speakers/headphones. I apologize that my naming sense is bad but it is what it is.

> **Current recommended version: AppMic v0.2.4 x64**

## Compatibility

AppMic was developed and tested around this exact setup:

- **Windows 11 x64**
- **Equalizer APO 1.4.2 x64**
- **Equalizer APO VST2 plugin filter**
- **AppMic v0.2.4 x64**

Other Equalizer APO versions may work, but they are not the configuration this project was built and tested against. AppMic depends on Equalizer APO's x64 VST2 hosting behavior, plugin lifecycle, audio-processing environment, and editor behavior, so older or newer versions can behave differently.

### Recommended Equalizer APO setup

**Use AppMic in Equalizer APO's Embed mode.**

That is the intended and most reliable way to run v0.2.4.

If you open AppMic in Equalizer APO's separate VST panel window instead, **do not press the Equalizer APO controls at the bottom of that window**:

- OK
- Cancel
- Apply
- Apply automatically

Those are host controls, not AppMic controls. During development they were observed to interrupt AppMic's live runtime state, stop capture/mixing, or require the VST filter to be disabled and enabled again. **Embed mode is strongly recommended.**

## Current stable binary

The known-good v0.2.4 binary is:

```text
AppMic-v0.2.4-x64.dll
Size: 104,960 bytes
SHA-256: 1a198c855314f6aa64d5a1eeb3cfee8d3bae70d66de186050ed17b195cd63515
```

The checksum is important. During development, another experimental DLL also carried the visible v0.2.4 name but was **98,816 bytes** and came from a different source lineage. It is not the recommended release.

## What AppMic does

- Select a running Windows application.
- Preview that application's audio before mixing it into the mic.
- Press **START** to mix the app into the microphone path.
- Press **STOP** to stop the app mix while keeping normal mic passthrough.
- Keep the application's normal speaker/headphone playback.
- Remember the selected application by executable identity rather than one temporary PID.
- Wait for the same application to return if it closes.
- Treat silence/pause as silence, not as the application disappearing.
- Show a source selector only when Windows exposes genuinely separate useful sources.
- Provide independent application gain.
- Show live **App Audio**, **Mic**, and **Output** meters.
- Keep microphone passthrough as the fail-safe path if app capture is unavailable.

## Installation

1. Install **Equalizer APO 1.4.2 x64** and configure it for the microphone/device you want to use.
2. Download **AppMic v0.2.4 x64**.
3. Put the DLL somewhere permanent, for example:
   ```text
   C:\Program Files\EqualizerAPO\VSTPlugins\
   ```
4. Open Equalizer APO Configuration Editor.
5. Select the microphone/device chain where AppMic should run.
6. Add **Plugins → VST plugin**.
7. Browse to `AppMic-v0.2.4-x64.dll`.
8. Enable **Embed** for the VST editor.
9. Use AppMic from the embedded panel.

## Basic use

1. Start the application whose sound you want to route.
2. Select that application in AppMic.
3. Confirm the **App Audio** meter reacts to playback.
4. Set **App Volume** as needed.
5. Press **START**.
6. The selected application's audio is mixed into the microphone path.
7. Press **STOP** when you no longer want the app mixed into the mic.

Selecting an app starts preview capture. START controls whether that already-previewed app signal is mixed into the microphone.

## Controls

### Application

Chooses which application's playback AppMic should capture.

If the selected app closes, AppMic can wait for the same executable to return instead of permanently tying the selection to the old PID.

### Audio source

Only appears when Windows exposes multiple meaningful capture sources for the selected application.

### App Volume

v0.2.4 supports **0% to 400%** application gain.

- **100%** = normal captured level
- **200%** = high-gain boundary
- **200–400%** = very high gain for unusually quiet sources

The slider marks 200% in red. Attempting to go above 200% shows a warning before high gain is allowed. The dedicated **Reset** button returns App Volume to **100%**.

### START / STOP

- **START** mixes the selected app into the microphone path.
- **STOP** removes the app mix while keeping microphone passthrough.

### Meters

- **App Audio** — captured audio from the selected application
- **Mic** — microphone input activity
- **Output** — final AppMic output before continuing through the Equalizer APO chain

The meters are activity indicators, not gain controls.

## Reconnect and fail-safe behavior

If the selected application closes:

- microphone passthrough stays available
- AppMic waits for the selected executable to return
- it checks periodically for a valid matching source
- it can reconnect when the app comes back

A silent or paused application is not automatically treated as closed.

If app capture fails, the microphone remains the fallback path.

## How it works

Early builds attempted process-loopback capture directly from Equalizer APO's processing environment. That exposed a Windows limitation: the Equalizer APO processing side can run in a service context where user-application process-loopback capture may return silent packets.

The architecture that ultimately worked moved capture into the logged-in user's session:

```text
Selected Windows application
          |
          | Windows process-loopback capture
          v
AppMic capture worker
(logged-in user session, launched from the same DLL)
          |
          | low-latency IPC / shared audio transport
          v
AppMic VST2 inside Equalizer APO
          |
          +---- microphone input
          |
          v
gain / sync / protection / mix
          |
          v
Equalizer APO microphone chain
```

The same AppMic DLL is used for both the VST and capture-worker roles; no separate custom AppMic EXE is required.

Later builds added adaptive buffering, fractional resampling for clock drift, smoother PI-style synchronization, startup prefill, underrun fades, output limiting, and bad-sample protection.

A diagnostic underrun count increasing occasionally does **not automatically mean audible corruption**. If the output sounds clean, the counter alone is not a reason to retune the audio engine.

## Troubleshooting

### AppMic does not load

Confirm:

- Equalizer APO is **1.4.2 x64**
- the AppMic DLL is **x64**
- the normal Equalizer APO VST plugin filter is being used

### App Audio meter does not move

- Make sure the selected application is currently producing audio.
- Select the application again.
- If an **Audio source** selector appears, choose the correct source.
- Restart the target application if necessary.
- Confirm you are using the recommended v0.2.4 binary/checksum.

### App Audio moves but is not in the microphone

- Confirm **START** is active.
- Confirm App Volume is not 0%.
- Confirm the VST filter is enabled in the correct microphone/device chain.
- Confirm Equalizer APO is installed/configured for that device.

### AppMic stopped after using Equalizer APO's bottom panel controls

Disable/re-enable the VST filter or restart the Configuration Editor, then use **Embed mode**.

For normal use, control AppMic through AppMic's own embedded UI.

### The selected application was closed

That is expected. AppMic should keep microphone passthrough available, wait for the selected executable to return, and reconnect when possible.

### Underruns increase but there are no audible pops

If the stream sounds correct, do not change the audio engine solely because the diagnostic counter increased. Stable later builds were observed to accumulate some diagnostic underruns without audible clipping or popping.

## Version history

This repository preserves the development line from the first loader test through the current stable v0.2.4.

| Version | Status | Main milestone |
| --- | --- | --- |
| **v0.1** | Historical | First x64 VST2 loader/UI validation. Process list and START/STOP UI worked; audio remained microphone passthrough only. |
| **v0.1.1** | Historical / important baseline | Cleaner application list, shorter scrollable dropdown, and preserved the known-loading passthrough baseline. |
| **v0.1.2** | Experimental | Added Windows audio-session discovery, per-process loopback capture, app mixing, reconnect behavior, conditional source selection, and a small safety buffer. |
| **v0.1.2-hotfix** | Historical | Rebuilt the capture work directly on the known-loading v0.1.1 structure while keeping the loader/static-import surface compatible. |
| **v0.1.4** | Development | Expanded live capture/mix behavior, app gain, metering, reconnect handling, and mic fail-safe behavior. |
| **v0.1.4-hotfix** | Experimental | Improved multi-process/browser target resolution, silent-target fallback, real-audio Streaming detection, and diagnostics. |
| **v0.1.4-hotfix-b** | Experimental | Reworked process-loopback format and target selection to diagnose persistent silent process capture. |
| **v0.1.4-hotfix-c** | **Major capture milestone** | Moved process-loopback capture into the logged-in user session using the same DLL via `rundll32.exe`. Real app capture and the App Audio preview meter began working. |
| **v0.1.4-hotfix-d** | Development | Added adaptive buffering, clock correction, soft limiting, custom meters/slider work, and reduced host-state churn. |
| **v0.1.4-hotfix-e** | Development | Added fractional adaptive resampling, more scheduling headroom, underrun fades, static UI behavior, and richer buffer/sync diagnostics. |
| **v0.1.4-hotfix-f** | **Major stability milestone** | Replaced the aggressive controller with smoother PI-style synchronization, tighter target buffering, startup prefill, and improved underrun handling. |
| **v0.2** | Historical | Product/UI pass around the Hotfix-F audio engine: Windows-style light/dark UI, limiter/clip diagnostics, per-app volume memory, smoother meters, reconnect/watchdog work, and DPI-aware presentation. |
| **v0.2.0** | **Scratched / not recommended** | Experimental branch abandoned after loader problems such as `Library could not be loaded`. Preserved only for development history. |
| **v0.2.1** | Good historical build | Returned to the Hotfix-F realtime ring-buffer/clock-sync path and separated UI telemetry from the realtime callback. |
| **v0.2.2** | Good historical build | Expanded App Volume to 0–400%, added high-gain indication/warning, state sanitization, status reliability work, saved-volume fixes, and host-state isolation changes. |
| **v0.2.3** | Good historical build | Reworked >200% authorization, removed the problematic double-click reset, added the dedicated volume Reset button, and kept the v0.2.1/Hotfix-F audio engine. |
| **v0.2.4** | **Current recommended stable build** | Current reliable release. Use the exact 104,960-byte binary/checksum documented above. |

## Historical source/build packages

Where the matching development package is available, the repository can preserve the exact package used at that stage. Those packages contain the source and build materials that were actually used, typically including files such as:

```text
src/AppMic.cpp
src/win32_min.h
src/coreaudio_min.h
src/vst2_min.h
build_windows.bat
BUILD_NOTES.txt
README.txt
compiled DLL / import library
```

The historical build line used x64 LLVM/Clang + `lld-link` style Windows builds and intentionally kept a small static import surface.

### v0.2.4 source status

The exact source snapshot corresponding to the recommended **104,960-byte** v0.2.4 binary has not been positively recovered.

A separate package labeled v0.2.4 builds a **98,816-byte** DLL from a different lineage. This repository therefore does **not** claim that package as the source of the recommended stable binary.

The stable v0.2.4 DLL is preserved by its exact SHA-256:

```text
1a198c855314f6aa64d5a1eeb3cfee8d3bae70d66de186050ed17b195cd63515
```

## Release policy

- **v0.2.4** is the current recommended release.
- Historical builds are kept for development history and regression comparison.
- Scratched builds are clearly marked and should not be treated as stable.
- The later v0.2.5 / Qt / Equalizer APO bottom-bar experiments are **not part of the official release line**.
- The official public history currently ends at **v0.2.4**.
- Release files should be verified using the SHA-256 checksums in this repository.

## License

AppMic source authored for this project is released under the **MIT License**. See [LICENSE](LICENSE).

AppMic is an independent project and is not affiliated with or endorsed by Equalizer APO, Microsoft, or Steinberg. Third-party names, APIs, SDK material, and licenses remain the property of their respective owners.

## Recommended setup

**Equalizer APO 1.4.2 x64 + AppMic v0.2.4 x64 + Embed mode.**
