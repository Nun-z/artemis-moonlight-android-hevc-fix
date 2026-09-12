# Artemis Android — Amlogic HEVC Fixes

A fork of [ClassicOldSong/moonlight-android](https://github.com/ClassicOldSong/moonlight-android)
(Artemis) that adds optional workarounds for HEVC decoding problems on Amlogic-based Android TV
devices, plus a build fix that matters for anyone streaming at high bitrates from a debug build.

Every fix is behind its own switch and every switch defaults to stock Artemis behavior. An
untouched install of this build behaves exactly like upstream Artemis.

Developed and tested on a **Homatics Box 4K Pro V2** (Amlogic S905X5M) streaming from Sunshine.

## The build fix: debug builds report phantom packet loss

This one is worth reading even if you don't care about Amlogic.

`Application.mk` sets no `APP_OPTIM`, and AGP passes `NDK_DEBUG=1` to ndk-build for debuggable
variants. `app/src/main/jni/moonlight-core/Android.mk` turns that into `-DLC_DEBUG`, and
`moonlight-common-c/src/RtpVideoQueue.c` reacts to it:

```c
#if defined(LC_DEBUG) && !defined(LC_FUZZING)
// This enables FEC validation mode with a synthetic drop
// and recovered packet checks vs the original input.
#define FEC_VALIDATION_MODE
#define FEC_VERBOSE
#endif
```

`FEC_VALIDATION_MODE` deliberately discards one packet per FEC block and rebuilds it from parity to
verify recovery, and it requires an extra parity shard before a frame counts as recoverable
(`neededPackets += queue->fecPercentage ? 1 : 0;`). When that parity isn't there, the frame is
reported through `notifyFrameLost()` — and the user sees it as packet loss. On top of that,
`LC_DEBUG` enables `assert()` in the per-packet path and `FEC_VERBOSE` logging, and the missing
`APP_OPTIM` drops the native code (including Reed-Solomon) to `-O0`.

Measured on the Homatics box over Gigabit Ethernet, 1080p60 at 80 Mbit:

| Build | Reported packet loss |
|---|---|
| Debug, stock | ~30 % |
| Debug, `NDK_DEBUG=0 APP_OPTIM=release` | 0.00 % |

Confirmed at 4K60 / 80 Mbit as well. This fork therefore forces release settings for native code in
debug builds. It is not an Amlogic issue and not an Artemis bug — it affects any debug build of any
Moonlight-derived client.

## The Amlogic fixes

All settings live under **Settings → Amlogic HEVC Fixes**. Changes take effect on the next stream.

### HEVC low-latency options

Controls which low-latency options are passed to the HEVC decoder.

| Value | Behavior |
|---|---|
| Default | Unchanged, full Artemis option ladder |
| No low-latency options | Only `KEY_PRIORITY=0`, safest |
| Test: KEY_LOW_LATENCY only | Only `low-latency` |
| Test: vdec-lowlatency only | Only `vdec-lowlatency` |
| Test: vendor.low-latency.enable only | Only `vendor.low-latency.enable` |
| vdec-lowlatency + vendor.low-latency.enable | Both, without `KEY_LOW_LATENCY` |

The three test modes enable exactly one option each, so you can find the one your firmware breaks
on. On the Homatics Box 4K Pro V2, `KEY_LOW_LATENCY` is the culprit and the combined mode is the
right choice.

Hooks into `setDecoderLowLatencyOptions()` before the vendor-specific blocks and only for MIME
`video/hevc`. On "Default" nothing is intercepted. If `configure()` rejects the selected option, the
decoder falls back to realtime priority only.

### Disable HEVC reference frame invalidation

Forces a full keyframe after packet loss instead of using RFI. Costs bandwidth when loss occurs, but
avoids the artifacts and decoder hangs some HEVC decoders show after RFI.

### Decoder stall watchdog

Detects the state where the decoder still accepts input but has produced no output for 5 seconds,
and restarts it through Moonlight's existing codec recovery path. Recovers from freezes and 0 FPS
without restarting the app.

Input-idle periods are excluded, so a host or network pause is never mistaken for a stall. There is
a 10 second cooldown between restarts, and watchdog restarts do not consume the `CR_MAX_TRIES`
budget reserved for real `MediaCodec` failures. Artemis' own C2 flush watchdog stays active and
triggers first; this is the escalation step when a flush is not enough.

### Non-blocking output queue

Two related renderer safeguards:

- Lets the decoder thread be interrupted while shutting down instead of blocking indefinitely in
  `take()` on a full output buffer queue. It keeps polling until a slot frees, so the queue never
  grows past `OUTPUT_BUFFER_QUEUE_LIMIT`.
- Keeps a single unexpected exception in `doFrame()` from breaking the Choreographer callback chain.
  `RendererException` is always rethrown, so the fatal-crash path is preserved.

With the switch off, both paths are byte-for-byte upstream.

### Force GPU composition (Android TV)

Mutates a 1sp transparent overlay on every Choreographer frame to keep the compositor active,
preventing some Android TV devices from handing the video layer straight to the display hardware.
The ticker is tied to the connection state and stops in PiP, `onStop()` and `onDestroy()`.

### Cosmetic and build

Debug builds are labeled "Artemis" instead of "Diana". The application ID suffix `.noirdebug` is
unchanged, so a debug build installs alongside an official Artemis rather than replacing it. ABI
splits are limited to `armeabi-v7a` and `arm64-v8a`; many Amlogic boxes, including the Homatics Box
4K Pro V2, run a 32-bit Android image and need the `armeabi-v7a` APK.

## Not included

The `moonlight-common-c` patch from the original Homatics fork (`notifyFrameLost()` setting
`waitingForNextSuccessfulFrame`, the "slideshow" fix) is not part of this fork. Artemis uses its own
`moonlight-common-c` submodule, so including it would require forking that repository as well.

Artemis's own defaults for resolution, codec and frame pacing are left untouched.

## Credits

The Amlogic fixes come from
[sven253/moonlight-android-homatics-box-4k-pro-v2](https://github.com/sven253/moonlight-android-homatics-box-4k-pro-v2),
which builds on the work of
[farnsworth3010](https://github.com/farnsworth3010/moonlight-android-mi-tv-stick-gen-2) and
[Viktsolovevwork278](https://github.com/Viktsolovevwork278/moonlight-android-hevc-fix) for the
Mi TV Stick 4K Gen 2 and Mi TV Box S 3rd Gen. Here they are rebuilt as device-independent,
individually selectable options — there is no device detection at all.

- [Artemis / ClassicOldSong](https://github.com/ClassicOldSong/moonlight-android)
- [Moonlight Android](https://github.com/moonlight-stream/moonlight-android)
- [Sunshine](https://github.com/LizardByte/Sunshine) and [Apollo](https://github.com/ClassicOldSong/Apollo)

## Building

Requires JDK 17, Android SDK 36, build-tools 36.0.0 and NDK 27.0.12077973.

```bash
git clone --recurse-submodules https://github.com/sven253/artemis-moonlight-android-hevc-fix.git
cd artemis-moonlight-android-hevc-fix
./gradlew assembleNonRoot_gameDebug
```

Or run the **Build APK** workflow from the Actions tab. It is manual-only (`workflow_dispatch`).

## License

GPLv3, same as upstream Artemis and Moonlight. See [LICENSE.txt](LICENSE.txt).
