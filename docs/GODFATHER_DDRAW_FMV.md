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

## Remaining work before marking the PR ready

- [Build completed] The clean `0076` patch compiled and was loaded (verified with `opacity=255/255`), but further runtime/regression validation is required.
- Confirm that the change to `device_parent_mode_changed()` avoids `Failed to resize window` and does not break fullscreen restoration.
- Verify multiple FMVs, black transitions, gameplay return, focus changes, resizing, and window destruction.
- Determine whether the afterimage coincides with overlay hide/close/reopen or occurs between ordinary overlay flips. Opacity 255/255 did not remove it.
- Check behavior with other display servers, compositors, resolutions, and WineD3D backends.
- Keep the workaround game-scoped or opt-in until broader compatibility is established; never replace the stock build solely on this evidence.

## Background

The preceding GE patch `0075-ddraw-emulate-one-visible-overlay.patch` supplies the basic software YUY2 overlay emulation. This patch builds on it instead of replacing its capabilities.

---

These corrections and their diagnostics were developed with assistance from artificial intelligence (AI).
