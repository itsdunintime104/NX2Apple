# Ocarina of Time on an M1 Max: state of the test work

Status on 2026-11-11. This page describes a private test tree, not the released app. Most of the settings named here are not in this repository's `main` yet. The table in "What the test tree holds" says which.

- Game: The Legend of Zelda: Ocarina of Time, title ID `0400486024DEC000`, update v131072.
- Machine: MacBook Pro with M1 Max, 32 GB, 120 Hz variable refresh screen, macOS 26.6.
- Test conditions from 11 Nov 02:25: power adapter connected, High Power mode, windowed, the same place in the game. Earlier runs were at least partly on battery. Their power state is not known, and that includes the 10 Nov baseline and the logged frame generation run.
- Goal: a steady 60 frames a second, or motion that feels like it.

## Where it stands

| Date | Frame time | Frames a second | Source |
|---|---|---|---|
| 10 Nov, start | 28.0 ms | 35.8 | run log |
| 11 Nov, run B | mean 18.7 ms, blocks of 300 frames from 16.7 to 20.9 ms | 53 | run log |
| 11 Nov, Render scale 0.75 | not logged | 40 to 56 | the tester read the HUD, and the log of that run was lost |

| 11 Nov, build 5, Render scale 1.0 | mean 18.4 ms over 3,600 frames | 54 | run log |
| 11 Nov, build 5, Render scale 0.75 | 16.7 to 17.4 ms in five blocks of 300 frames, about 25 s of play | 60 | run log |

Build 5 is the test tree with pull request 1929, the early depth tests and the shared command allocator. The table "What the test tree holds" lists them.

60 frames a second needs 16.7 ms. At the game's own resolution the game does not hold it yet. Two faults remain:

- Hitches of 95 to 204 ms, a few each minute.
- Uneven motion below 60. The game moves in whole steps of 1/60 s. A late frame is paid back later by a double step. In run B 514 of 1,800 frames took a double step, and each was shown about as long as a single one. The average speed is right (1.000 of real time), but the motion speeds up and slows down. After a hitch the debt is paid as a burst, which looks like fast forward.

## Settings used in the test build

| Setting | Value | Why |
|---|---|---|
| Frame rate | 60 | the game's own target |
| Game's upscaler | MetalFX | replaces a compute chain of the game that cost 3.2 ms a frame |
| Render scale | 0.75 | draws the 3D views smaller. The HUD still says 1440p. The game composes its final picture at 2560 x 1440, and the scale changes only the 3D views |
| Encode thread | On | Encode ahead needs it |
| Encode ahead | On | translates the next frame while the GPU still draws the last one |
| Frame generation | Off | experimental, see below |
| GPU overlap | Off | the picture flashes with it on (see "The model") |

Frame generation is in the test build and is not ready:

- 30 + 30 holds the game at 30 and MetalFX makes one frame between two. On the adapter in High Power the tester called it "amazing". It has no logged numbers. It seems to turn off during talk with a character, and the cause is not known.
- Uncapped, one logged run on 10 Nov at a 1080p output: real frames fell from about 41 to between 32 and 36 a second, 65 to 72 were shown, and MetalFX cost 4.1 ms a frame. The tester saw 70 to 80 on the HUD and said it was not smooth and had some input lag.
- Open faults from the first runs: the picture shakes at the border between the scene and the HUD, and some generated frames show ghosts.
- Since 11 Nov the Uncapped mode also holds late frames, so the game steps evenly at 60 or 30.
- Since 11 Nov frame generation runs with the encode thread. One run: 300 of 300 frames interpolated, no generated frame dropped, clean exit. MetalFX cost 3.7 to 3.8 ms a frame there.
- Uncapped at Render scale 1.0 (build 5): the game fell to 31 real frames a second and the tester saw ghosts. The generated frame costs GPU time, and the real frame was already at the limit.
- Uncapped at Render scale 0.75 with the encode thread: 60 real frames were not held. Six tries at a hold of one refresh failed with a real frame of 17.0 to 19.4 ms, and it fell back to 30 + 30.
- MetalFX makes one frame between two real frames. On a fixed 120 Hz link only 60 + 60 and 30 + 30 give even spacing.
- The tester rejected Render scale 0.75 as the answer: the picture is visibly softer. The target is the game's own resolution.

## What the test tree holds

"In main" means this repository's `main` at engine pin `ab41d6a`.

| Change | Source | In main | Measured here |
|---|---|---|---|
| 120 Hz display link (`DM_PRESENT_LINK=2`) | engine switch | yes | part of the gain from 28.0 to 18.7 ms, not separated |
| Shaders read global memory at Metal's own addresses | upstream pull request 1833, open | no | GPU time 22.66 to 20.62 ms a frame, run log |
| MetalFX takes the game's own upscaler | upstream pull request 1951, open | no | removes a 3.2 ms compute chain, run log |
| Render scale for this game | upstream pull request 1903, open | no | 40 to 56 frames a second at 0.75 by the tester, no log |
| Border corrections compiled out of pipelines that do not need them | upstream pull request 1929, closed | no | not measured here. Upstream reports 1.8 ms less on an M4 Air |
| Early depth tests from the shader's own flag | upstream commit `a888e668` | yes | not measured here |
| One command allocator shared by a frame's submissions | upstream commit `fd8188c9` | no | in build 5: no allocator wait, and hitches remain (2 in 3,600 frames). So the allocator pool was not their cause |
| Encode ahead (`DM_NVN_PREENCODE`) | written in the test tree | no | part of the gain from 28.0 to 18.7 ms, not separated |
| Frame generation (`DM_MFX_INTERP`) | adapted from MacNativeGames (MIT license) | no | one logged run of the Uncapped mode, see "Settings used in the test build". Not ready |
| GPU overlap (`DM_NVN_SYNC_LAG`) | written in the test tree, experiment only | no | build 5 at Render scale 1.0: 18.1 ms against 18.4 ms, and the picture flashed. Stopped: the gain is under the 1.5 ms gate |
| Frame generation with the encode thread | written in the test tree | no | see "Settings used in the test build" |
| Copy seam switch (`DM_NVN_COPY_SEAM`, off by default) | written in the test tree, experiment only | no | never acted in the test run. See "Results of 11 Nov" |

What did not help, each measured: fewer or merged submissions, the encode thread on or off, the GPU clock held at its top frequency, GPU overlap, and the copy seam switch.

The changes written in the test tree on 11 Nov are in [`experiments/ocarina-m1-max`](experiments/ocarina-m1-max/README.md) as diff files. They do not apply to this repository's engine pin.

## Results of 11 Nov (build 5, the game's own resolution)

Where the GPU time of one frame goes, from the host's per-pass profile of four frames:

- The CPU has encoded the frame after 7.6 ms. The GPU ends it after 15.9 ms and works the whole time. The GPU is the limit.
- The frame has about 110 draw passes and about 30 compute or copy steps. The per-pass timer starts after a pass's vertex stage. 11.4 ms lie inside the timers, and 4.5 ms lie before passes start. That part is most likely vertex and mesh work that waits for earlier work. Upstream's trace on an M4 Pro shows the same split.
- About 30 draw passes a frame start behind a full wait: about 25 after a compute or copy step, about 5 after a barrier of the game. The other 80 start without it.
- Read in the game's code: after every compute dispatch and every texture copy the game asks that later vertex work waits. It is one generic pattern of the engine. So these waits are the game's request, and the host may drop one only with its own proof that the next pass does not read what was written.

The copy seam switch tried that for passes that follow only texture copies. It failed:

- In the test run it never acted. A dump of one frame shows why: the game makes 2 texture copies a frame, and of 99 gaps between passes only 1 holds nothing but a copy. 50 hold only barriers and state changes, 33 hold uniform data updates, and 17 hold compute dispatches.
- The assumption behind the switch (most copy steps between passes are texture copies) was not checked against such a dump before the switch was built.
- The same run compared "every pass waits" with the normal state: 18.4 ms against 17.5 ms as the median of 15 phases each. Two states that ran identical code differed by 1.0 ms, and the scene moved during the run. So this run does not show whether overlap of vertex and fragment work shortens the frame on this GPU.

Later the same day, from the same data and three more runs:

- Overlap does help. The profiled frames of that run give the GPU's time for one frame directly, and two states that ran identical code agree within 0.15 ms: 18.1 ms with every pass waiting, 15.9 and 16.1 ms in the normal state. The 80 passes a frame that start without the full wait save about 2.1 ms, about 0.03 ms each.
- The dump was read before the next build. In 15 of the 17 gaps with compute dispatches, the passes that follow do not read what the dispatches wrote before their fragment stage.
- The switch was extended to those gaps: a dispatch's storage buffers and images join the list that each draw is checked against. In a run of 35 phases of 5 seconds it opened about 5 gaps a frame, and a draw closed one again in 0.1 to 0.5 frames. The GPU's time for one frame was 16.8 ms with it off and 17.2 ms with it on (means 17.07 and 17.06 ms). No measurable gain, which fits 5 gaps at 0.03 ms. The tester saw no picture fault.
- What keeps the other gaps closed, per frame: a copy step that holds only uniform updates or host copies (13 to 14), an ordering barrier of the game from before the first write (6), a draw that writes memory from its shader (4). Opening all of them would be worth about 0.7 ms at most, with a rising risk for the picture.
- The host's pass merging (`DM_NVN_TBDR=1`, off for this game) was tried once: no gain, more long frames, and wrong water. It stays off.
- The fragment shaders of the costly passes were inspected for slow translation. They are large (3,000 to 6,000 lines, 16 to 23 texture reads), and no fault of that kind was found.

**Decision of the tester, 11 Nov:** the game's own resolution at about 57 frames a second (17.4 to 17.5 ms in the test scene) is the accepted baseline. A lower render scale and dynamic resolution are rejected, because the picture comes first. The work on 60 at this resolution stops here.

Not found yet: the cause of the hitches. In play, a hitch is one command buffer that the GPU holds for 20 to 150 ms, with no upload, no new pipeline, no allocator wait and no pattern in the residency numbers. So the cause is not visible from the game or the host. Other apps that use the GPU and swap are suspects, not tested yet.

## The model

1. The frame is a serial chain. The game builds a frame. A host thread translates it (7 to 9.5 ms of CPU). The GPU runs it (15.4 to 20.4 ms from the first start to the last end). The game then waits until the GPU is done before it lets the next frame's commands run. In run B that wait took 3.7 to 11.5 ms a frame.
2. The GPU is not saturated. Holding its clock at the top did not change the time. The cost is latency in series: 111 to 115 render passes and 28 to 37 submissions a frame, and about 71 of the passes load and store whole attachments. A tile-based GPU pays at each pass boundary.
3. The wait in point 1 is the game's own code. Each frame it submits with a fence, acquires the next window texture, waits on that fence with no timeout, and only then signals the event that the next frame's commands wait on. The game never lets one frame start while the GPU holds the frame before it. So nothing in the game promises that a frame's memory stays untouched while the next one is built. The likely cause of the flashing with GPU overlap follows from this: the host lets the game run ahead, and the game rewrites memory that the GPU still reads. Which memory that is has not been measured yet.
4. The hitches fall in the blocks of run B where the host's pool of 32 command allocators ran out. This is a candidate cause, not a proven one.

## Next steps

Steps 1 and 2 of the earlier plan are done (build 5 measured, GPU overlap stopped). What is next:

1. Play tests for stability with the baseline settings: crashes, picture faults, hitches.
2. The hitch cause: a play run with no other app on the GPU, and a count of the long frames.
3. More tests of what the game's code shows, each as a quick switch that is checked against a one-frame dump before it is built.
4. Even motion: 60 + 60 frame generation needs the real frame under 16.7 ms, so it is parked. 30 + 30 is stable.
5. Bring the changes of the table into this repository as patches, each with its measurement. Until then, of the settings in the table "Settings used in the test build", the released app shows only Frame rate and Encode thread for this game. Its Upscaling row is a different setting from Game's upscaler.

## Plan for reverse engineering

The game's code is read with REA, an MCP server that drives Ghidra.

Rule: measure with a run first. Read the game's code only when a log cannot answer the question and the answer changes what gets built.

What works:

- The whole game module is too large for the tool's start limit. One function at a time works and takes seconds.
- So each question is narrowed first with small scans (a table of the game's NVN function pointers, then the call sites of one function), and only the functions that the scans name are decompiled.

Done so far:

- Point 3 of the model. It confirmed a measurement, named its parts, and showed that the game has no usable switch that removes the wait. The one mode that skips it also skips the window.
- The game keeps two sets of per-frame buffers and one main command buffer.
- The barrier the game issues after each compute dispatch and texture copy, and the engine features that make texture copies (fog pre-pass depth, soft-edge depth, a shared copy helper for snapshot, picking and exposure). See "Results of 11 Nov".

Planned, in this order, each behind its own gate:

| Question | Gate that must pass first | Why the game's code is needed |
|---|---|---|
| Which memory does the game rewrite in place each frame? | step 2 above shows a gain | a host diagnostic names the ranges first. The code is read only to explain a range the diagnostic names |
| Where is the frame step computed and used, and does the game accept a step of 1.5? | a decision to build the 40 frames a second step | the step lives in game code and no log shows where |
| Where do the 28 to 37 submissions a frame come from? | a GPU trace shows that pass boundaries are the cost | only the game's code shows why it flushes |

Not planned: analysis of the whole module, more tooling, or reading code to confirm what a run already measured.

Handling: decompiled code stays on the test machine. Only findings about behaviour are written down. No game file, key or extracted data is committed.
