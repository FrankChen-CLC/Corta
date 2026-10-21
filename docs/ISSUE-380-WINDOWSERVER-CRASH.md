# WindowServer crash when Corta launches on macOS 26.2 (#380)

## What failed

The supplied `../error_info` reports a crash of **WindowServer**, the macOS
process that composites windows and drives displays. It is not a Corta crash
or a kernel panic. When WindowServer aborts, the displays go black and the
user session returns to the login window. This explains the visible failure.

The supplied report is from a Mac mini M4 Pro (`Mac16,11`) running macOS 26.2
(`25C56`). Its main thread raised `EXC_CRASH (SIGABRT)` in an assertion at
`CA::OGL::MetalContext::stop_encoders`, called from `add_shared_event`,
`retain_image`, `detach_contents`, and `render_contents_background`, then
SkyLight's `MetalCompositeLayer`. It shows three threads named
`coreanimation.display.external-0/1/2` and `consecutiveCrashCount: 3`.
**These thread names do not establish the number of connected monitors.**
The reporter says the Mac has one monitor, and a current
`system_profiler SPDisplaysDataType` check lists one online display.
Near the first crash, ToDesk's display query in the unified log repeatedly
listed a single active display (`0x00000002`, 3008 × 1692 points). This
supports the reporter's correction for the incident itself. Issue #380's
"three external displays" description should be corrected; it inferred a
monitor count from thread names.

Four other unique WindowServer reports present on this Mac from 19 October
2026 have the same first compositor
frames and counts 1, 2, 4, and 5. One additional file duplicates count 1.
The unified log also records Corta processes starting at 18:10:50 and
18:11:24, before the 18:11:31 crash report was captured. Those timestamps
support the reported launch association, but do not identify the surface.

Thus the immediate cause is Apple's QuartzCore compositor aborting while
handling a Metal surface and its synchronization event. A client window
should not terminate WindowServer; the assertion belongs to macOS. Corta
appears to trigger it on launch, according to [issue #380], but these
WindowServer reports contain no Corta window identifier or definitive
render-path attribution.

## Which Corta path is implicated

Corta has two ways to present a frame:

1. In an ordinary window, `Metal4Backend.endFrame` commits a Metal 4 command
   buffer that wrote the drawable, then calls `waitForDrawable` and
   `signalDrawable`. WindowServer must composite the resulting surface.
2. In a full-screen window with exactly one pane, Corta 1.2.x normally
   renders into an offscreen staging texture, then uses a classic Metal
   command queue to copy into the drawable and present it. Two
   `MTLSharedEvent`s synchronize the queues. This was measured only on
   macOS 27 with one built-in panel.

The original issue proposed the second path as a trigger. Further local
inspection found **no saved state or config for the installed release** in
`~/Library/Application Support/Corta` or `~/.config/corta/config` at the
time of this investigation. If that was also true during the crashes, Corta
launched a fresh window, so the first path was active and the full-screen
path was not. The absence of a file now is not proof of its absence at crash
time. The repeated stack also cannot distinguish which drawable WindowServer
was compositing. Nevertheless, avoiding only the full-screen path would
leave a plausible fresh-window crash untouched.

## Code change

On macOS 26, Corta now renders **every terminal window** into an offscreen
Metal 4 texture and copies it into the drawable with a classic Metal queue.
The drawable is presented by the classic command buffer. The
`CAMetalLayer` is made copyable before the first frame, and the policy is
applied to every pane even when `CORTA_DIRECT_FULLSCREEN=0`. macOS 27 and
later retain the existing direct full-screen optimization.

This routes around the suspected Metal 4 drawable presentation path. It is
still a hypothesis-driven mitigation: the classic path also uses shared
events between queues, so the available reports do not prove that it will
avoid Apple's assertion. The reporter has since confirmed that the
diagnostic build runs normally on the affected one-monitor setup. This is
encouraging, but longer use is needed before calling the crash fixed.

## Verification and remaining test

The repository policy test covers an ordinary and a full-screen window on
macOS 26 and the old behavior on macOS 27. A standalone Swift compile and
run of that policy passed on this Mac. The normal full app build did not
complete: this checkout pins Xcode 26.6, but the installed Xcode is 26.2
and rejects existing `nonisolated lazy` properties in the renderer. An
isolated worktree containing the exact presentation change did build with
Xcode 26.2 after temporary compiler-compatibility edits and a Swift 5
language-mode override. The build alone does not validate the pinned Swift 6
configuration; the reporter subsequently launched it on the affected setup.

The reporter says the diagnostic build runs normally. No new WindowServer
crash report appears after the 19 October reports at the time of this check.
The direct-presentation policy and Metal backend tests passed (24 tests),
including copied pixels, frames in flight, and stalled GPU recovery. The
seven optimized renderer performance tests also passed. On this Mac, an
alternating headless benchmark into a 2940×1846 texture measured 60 full
rebuild frames per path: direct 0.358 ms average versus copy 0.555 ms average
to GPU completion. This roughly 0.20 ms per frame cost does not measure
visible latency or power. One extra staging texture is retained per pane,
about 21.7 MB at that size. `docs/PERFORMANCE.md` §5.17 has the numbers.

The wider Debug unit run exercised 1,079 tests. Four pixel-reference checks
each differed by 22 bytes; the same four failures occurred with the original
source on this Mac, including its direct and copy render paths. One
frames-in-flight test failed during the first parallel run, then passed
alone and during a second parallel run. The second run's test runner did not
exit after reporting its suites, so it was stopped. These checks found no
repeatable function regression from the change, but the full suite is not
green under this Xcode 26.2 compatibility build.

A build with Xcode 26.6 and a longer affected-Mac run still need verification.
That run should cover a fresh window, split panes, full screen, quit and
restore, and sustained output without a new WindowServer report. If the
crash persists, capture the new report and test a rendering fallback that
does not use the Metal 4 queue or cross-queue shared events.

The assertion itself should be reported to Apple through Feedback
Assistant, attaching the WindowServer reports and display configuration.

[issue #380]: https://github.com/noah-qin/Corta/issues/380
