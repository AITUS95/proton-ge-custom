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

The alpha result was obtained from an experimental locally compiled Wine DLL. The cleaned-up standalone `0076` patch in this PR is reconstructed from the tested approach and **has not yet been independently built or regression-tested**.

## Remaining work before marking the PR ready

- Compile and test the clean `0076` patch on top of the current GE video patch series.
- Confirm that the change to `device_parent_mode_changed()` avoids `Failed to resize window` and does not break fullscreen restoration.
- Verify multiple FMVs, black transitions, gameplay return, focus changes, resizing, and window destruction.
- Investigate the small right-edge flash on fade-to-black.
- Check behavior with other display servers, compositors, resolutions, and WineD3D backends.
- Keep the workaround game-scoped or opt-in until broader compatibility is established; never replace the stock build solely on this evidence.

## Background

The preceding GE patch `0075-ddraw-emulate-one-visible-overlay.patch` supplies the basic software YUY2 overlay emulation. This patch builds on it instead of replacing its capabilities.

---

These corrections and their diagnostics were developed with assistance from artificial intelligence (AI).
