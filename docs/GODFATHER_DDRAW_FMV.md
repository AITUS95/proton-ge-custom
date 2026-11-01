# The Godfather (2006): experimental DirectDraw FMV overlay workaround

This document tracks a **work-in-progress** Wine DirectDraw workaround for the 2006 PC release of *The Godfather* on GE-Proton. It is **not enabled by default**.

## Problem

The game uses YUY2 DirectDraw overlay video (640×480, displayed at 800×600) while DXVK also presents D3D9 frames. The two presentation paths can interfere, causing black-frame flickering in the FMVs.

## Opt-in workaround

Enable both environment variables:

```sh
GF_DDRAW_SEPARATE_WINDOW=1 GF_DDRAW_ALPHA_TEST=1 %command%
```

The Wine patch creates a dedicated topmost DirectDraw popup for the active overlay, with `WS_EX_LAYERED` and alpha **254/255**. It rebinds the DirectDraw swapchain to the transient window, restores the original target when the overlay is hidden or destroyed, and avoids resizing the transient popup during an exclusive-mode notification.

The second variable remains named `GF_DDRAW_ALPHA_TEST` because this is an experimental implementation; the two variables together are required.

## Manual diagnostic results

Tested locally on Ubuntu 26.04, NVIDIA RTX 5080 Mobile, XWayland/Wayland, and a GE-Proton11-11 diagnostic build (32-bit Wine DirectDraw DLL).

| Experiment | Observed result |
| --- | --- |
| Original concurrent DirectDraw/D3D9 presentation | Black-frame flickering |
| Separate opaque DirectDraw window | Flickering disappears, but FMV falls to ~1 FPS for 10–14 seconds |
| Separate window with a 16-pixel exposed region | Smooth ~30 FPS video, but a right-side visual band |
| Separate window with a 1×1-pixel hole | Smooth video, but the same band |
| Separate window without the region modification | Correct geometry, but slow video |
| Separate layered window, alpha 254/255, no region modification | Smooth, centered video, no broad flicker or side band; minor right-edge flash on transitions to black |

In the slow case, both D3D9 Present and DirectDraw UpdateOverlay dropped to about one call per second, then recovered to roughly 30/s. The modified region avoided the slowdown but caused an unrelated-looking border artifact. The alpha implementation avoided both symptoms during the observed movie playback.

### Remaining transient on black fades

The user highlighted an intermittent, short-lived **right-edge afterimage** visible as the movie fades to black, resembling a fragment of a previous frame. This is **not** the persistent 10% right-side band from the region-based experiments; the centered image otherwise displays normally with the layered-window path.

The cause is not established. Two competing hypotheses require an A/B check: (1) a tiny amount of stale D3D9 imagery bleeds through the 254/255 layered alpha; (2) WineD3D briefly displays stale edge pixels in its frontbuffer. The experimental patch exposes `GF_DDRAW_OPAQUE=1` (255/255 alpha) as a **diagnostic only**. It may bring back the previously observed ~1 FPS throttling, and must not be treated as the final fix. **Observed on 2026-10-31:** With `GF_DDRAW_OPAQUE=1`, diagnostic logging confirmed the popup at `opacity=255/255` on two FMV openings. The user reported both **renewed slow playback** and the **same right-edge flash**. The tested log excerpt contained **no** `device_parent_mode_changed: Failed to resize window` errors, although this is not full lifecycle validation.

This falsifies *simple alpha bleed-through as the sole explanation* for the afterimage. The next diagnostic target is whether the flash aligns with an overlay-window close/reopen boundary or occurs during normal overlay flips. Check `GF-DDRAW-WINDOW: closed`, `GF-DDRAW-WINDOW: opened`, and `ddraw_surface7_UpdateOverlay` around the visible transition before attempting another rendering patch.

Do not change video cropping, overlay rectangles, or use `SetWindowRgn()`: these experiments were unsuccessful or produced new rendering artifacts.

### Overlay transition trace in the opaque test

A subsequent read of the same `GF_DDRAW_OPAQUE=1` log showed:

| Event | Wine log timestamp |
| --- | ---: |
| `DDOVER_HIDE` (`UpdateOverlay flags 0x200`) | 48013.680 |
| `GF-DDRAW-WINDOW: closed` | 48013.684 |
| Next `GF-DDRAW-WINDOW: opened` | 48014.176 |

There was an approximately **492 ms period without the overlay popup**. This makes window lifecycle / exposure of the underlying D3D9 surface a reasonable investigation path, but **there is no timestamp for the visible flash**, so it remains unproven that the flash occurred during that interval. Avoid interpreting `DDOVER_HIDE` as a failure: it is an expected overlay lifecycle event.


The initial alpha result was obtained from an experimental local Wine DLL. The cleaned-up standalone `0076` patch **was subsequently compiled and loaded** in the diagnostic Proton environment: its `GF-DDRAW-WINDOW: opened ... opacity=255/255` markers were observed with `GF_DDRAW_OPAQUE=1`. This verifies that the newer implementation built and ran, but **does not establish that the layered 254/255 path or lifecycle fixes have passed all regression tests**.

## Offline VP6 comparison (user-supplied image sheets)

The game installation's `Paramount.vp6` was decoded outside Wine using FFmpeg and two contact sheets were shared:

- `paramount_panoramica.png`: samples progress through the Paramount logo's fade-to-black without an obvious persistent right-edge vertical afterimage.
- `paramount_ultimi_30_frame.png`: its entire RGB bitmap is black (every pixel is `0,0,0`), consistent with a clean all-black tail.

This strongly favors an artifact in the **video presentation path** rather than an intentional visible stripe encoded into the intro. However, a tiled contact sheet is not a pixel-by-pixel inspection of **every** decoded VP6 frame, nor does it prove where the stray pixels originate. A precise compositor capture aligned to the flicker or a raw/decoded frame check remains useful.

The user reports the narrow right-edge trace **starting during the Paramount fade**, then briefly flashing against the black screen before the Electronic Arts title. Therefore a correction that *only* runs on `DDOVER_HIDE` might be too late: the suspect pixels may already have appeared during ordinary overlay updates.

**Do not introduce an arbitrary edge crop or black stripe:** first establish whether the pixels appear in the overlay source, the WineD3D destination before presentation, or the compositor's final output. Keep the proven 254/255 window behavior unchanged.

## Measured capture evidence: two-frame right-edge flash

A real-time 60 FPS gameplay recording, supplied as `Replay_2026-11-01_01-03-46.mp4`, was inspected frame by frame. Format: H.264, 2560×1600, 1,243 decoded frames, duration about 20.72 seconds.

The Paramount logo fades from approximately 15.3 to 16.2 seconds. The screen is fully black for a long interval, then the following **two consecutive video frames** show an isolated border artifact:

| Frame (zero-based) | Timestamp | Observation |
| ---: | ---: | --- |
| 1085 | 18.0833 s | Completely black |
| **1086** | **18.1000 s** | Full-height vertical line at x=2344–2346 |
| **1087** | **18.1167 s** | Identical full-height vertical line |
| 1088 | 18.1333 s | Completely black again |
| 1091 | 18.1833 s | Electronic Arts text starts appearing |

The measured stripe is exactly **3 screen pixels** wide, spanning y=0–1599. Each affected column is **uniform across all 1,600 rows**, with decoded pixel values in BGR approximately (26,29,27), (116,119,117), and (13,16,14), respectively. All other pixels in the two affected frames are black. The artifact lasts about **33.3 ms** at 60 FPS.

The evidence supports a **transient window/compositor edge** or a presentation boundary problem much more strongly than a colorful residual VP6 image. A gray solid line that stretches across the *entire* monitor is not characteristic of a stale Paramount frame. The capture does **not** by itself prove which Wine/XWayland/X11 event causes the line.

**Next discriminating step:** correlate the video timestamp of the second overlay with `GF-DDRAW-WINDOW: opened` and `closed` entries from the **same game run**. Previous logs from different runs must not be assumed temporally aligned with this capture. Then instrument or adjust popup mapping/presentation order only if correlation supports it. Preserve default layered alpha 254/255 and keep the 255/255 diagnostic disabled.

## Remaining work before marking the PR ready

- [Build completed] The clean `0076` patch compiled and was loaded (verified with `opacity=255/255`), but further runtime/regression validation is required.
- Confirm that the change to `device_parent_mode_changed()` avoids `Failed to resize window` and does not break fullscreen restoration.
- Verify multiple FMVs, black transitions, gameplay return, focus changes, resizing, and window destruction.
- Correlate the precisely measured two-frame 3px border flash at 18.100–18.117 s with window mapping/show and overlay lifecycle events from the same run; the previous opaque test did not eliminate it.
- Check behavior with other display servers, compositors, resolutions, and WineD3D backends.
- Keep the workaround game-scoped or opt-in until broader compatibility is established; never replace the stock build solely on this evidence.

## Background

The preceding GE patch `0075-ddraw-emulate-one-visible-overlay.patch` supplies the basic software YUY2 overlay emulation. This patch builds on it instead of replacing its capabilities.

---

These corrections and their diagnostics were developed with assistance from artificial intelligence (AI).
