# Ocarina of Time on an M1 Max: experimental host changes

These are diff files from a private test tree. They are a record, not patches of this repository:

- Base: seedds/nxnative at `04811a63` with the open pull requests 1833, 1951, 1903, 1929, 1846 and 2090 applied, plus a local port of MetalFX frame interpolation (`nvn_mfx.m`).
- They do **not** apply to this repository's engine pin (`ab41d6a`), and they are not in `patches/series`.
- Bringing them in as real patches is step 5 of the plan in [`OCARINA-M1-MAX.md`](../../OCARINA-M1-MAX.md).

| File | What it does | State |
|---|---|---|
| `01-crash-fix-and-hitch-diagnostic.diff` | Frame generation's UI shader copies are specialized with an empty constant set, so Metal's validation no longer aborts on a shader that declares function constants (pull request 1929 adds one). A log line `[slowcb]` names a command buffer the GPU held for 20 ms or longer. | Crash fix proven by a run. The diagnostic has printed only scene-load lines so far. |
| `02-frame-generation-with-encode-thread.diff` | The interpolated frame's command buffer is committed through the record stream, in order with the frame's other commits. The presenter shows a generated picture only when its commit has reached the queue. | Proven by one run: 300 of 300 frames interpolated, none dropped, clean exit. |
| `03-copy-seam-switch.diff` | `DM_NVN_COPY_SEAM` (off by default): a draw pass that follows only the game's texture-to-texture copies starts its vertex work without waiting for them. Each draw is checked, and a draw that may read GPU-written memory before its fragment stage puts the wait back. | Did not help. It never acted in the test run: the game makes 2 such copies a frame. Kept as a record of the approach. |

No game file, key, address of game code or extracted data is in these files.
