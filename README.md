# Technical Audit Report

## Scroll-Onset Main-Thread Stall Attributable to the Dynatrace OneAgent Jetpack Compose User-Interaction Instrumentation

| Field | Value |
|---|---|
| Document identifier | DT-COMPOSE-AUDIT-2026-001 |
| Document status | Issued for external audit |
| Revision | 1.0 |
| Date of issue | 2026-08-26 |
| Subject component | Dynatrace OneAgent for Android, version **8.341.1.1004**, applied through the Dynatrace Android Gradle instrumentation plugin (`com.dynatrace.instrumentation`) |
| Affected framework | Jetpack Compose UI (`androidx.compose.ui` / `compose-foundation` 1.10.0; Compose Multiplatform 1.10.0) |
| Investigation period | 2026-07-03 to 2026-07-16 (causal closure) |
| Evidence repository | A dedicated Git repository containing the hypothesis register, evidence bundles and tooling (referred to below as *the evidence repository*). Identifiers of the applications, organisations and hosts involved are withheld from this document. |

---

## 1. Purpose

This report presents, for independent external review, the investigation into a main-thread stall that occurs at the onset of a scroll gesture in Android applications built with Jetpack Compose and instrumented with the Dynatrace OneAgent. It states each finding together with its supporting evidence, assigns each finding an explicit evidence grade, records every hypothesis that was tested and rejected — including hypotheses advanced by the investigators that proved incorrect — and enumerates the questions that remain open.

The report is descriptive. It does not recommend a course of action to any party.

## 2. Scope

**In scope.** The behaviour of the OneAgent's Jetpack Compose user-interaction instrumentation at pointer-event time: its mechanism, its cost model, its dependence on UI topology, device class, input method and agent configuration, and the mitigations that were exercised.

**Out of scope.** Application-specific defects unrelated to the Agent (one such defect, a scroll-steal in the host UI library, was identified during the same period, corrected separately and excluded from all measurements reported here); the security posture of any application; and any assessment of the Dynatrace product beyond the specific behaviour measured.

**Confidentiality.** The identities of the production application, of the organisations involved, of the instrumentation tenant and of the build infrastructure are withheld. Applications are designated as follows:

- **Application A** — a production Android application, third-party to the investigators, obtained as a release/debug APK and analysed by decompilation (`jadx`) and bytecode patching. It embeds a Compose-based UI library and the Dynatrace OneAgent 8.341.1.1004.
- **Application B** — an independently developed Compose application, built from source by the investigators, embedding the same Compose-based UI library, instrumented with Dynatrace plugin 8.343.1.1038 against a dummy beacon. It can render the same content either inside a WebView or natively in Compose.
- **Reproducer** — a minimal Kotlin Multiplatform + Compose Multiplatform application written from scratch by the investigators, instrumented with Dynatrace plugin 8.341.1.1004 against a dummy beacon, containing no third-party UI library.

## 3. Definitions and abbreviations

| Term | Definition |
|---|---|
| **Agent** | Dynatrace OneAgent for Android, the runtime library woven into an application by the Dynatrace Gradle plugin at build time. |
| **Weaving** | Build-time bytecode transformation performed by the Dynatrace Gradle plugin (ASM transform task `transformDebugClassesWithAsm`). |
| **Hook** | The single call to `com.dynatrace.agent.compose.userinteraction.sensor.UserInteractionsApi.onNodeCoordinatorHit(LayoutInfo, boolean, List)` inserted by weaving into `androidx.compose.ui.node.NodeCoordinator.hit`. |
| **Hit test** | The Compose framework's per-pointer-event traversal that determines which layout nodes receive the event. |
| **N** | The number of attached Compose layout nodes whose window bounds intersect the rectangle passed to the Agent's `getOverlappedLayouts`; the operative size parameter of the Agent's per-touch work. |
| **Pass** | One complete execution of the Hook and its downstream computation for a single pointer event. |
| **DTSTALL** | A main-thread watchdog instrumented into the Reproducer; reports the peak main-thread stall ≥ 80 ms per gesture and whether a Dynatrace frame was on the stalled stack (`dyn=true`). |
| **DT_PROBE** | Logging instructions injected (smali) into the Agent's `TouchUserInteractionHandler`, reporting N and the wall-clock duration of `calculateTouchInformation` per pass (`calc_ms`). |
| **schedstat** | `/proc/<pid>/task/<tid>/schedstat` — kernel accounting of CPU time consumed by the main thread; used to distinguish computation from waiting. |
| **Test device** | A mid-range arm64 Android 13 handset with a 120 Hz display (single physical device used throughout). |
| **Test emulator** | Android Emulator, x86_64 system image, API 35 (Android 15), windowed, hardware GPU. |

## 4. Evidence grading scheme

Every finding in Section 6 carries one of the grades below. The grade characterises the **quality of the supporting evidence**, not the investigators' degree of belief.

| Grade | Designation | Criteria |
|---|---|---|
| **G1** | Demonstrated | Controlled comparison varying a single factor; reproduced across at least three repetitions or at least two independent environments; artefacts pinned by SHA-256; where a dispute existed, the verdict criteria were fixed in writing before data were collected. Reproducible by a third party from the pinned artefacts. |
| **G2** | Measured, not independently replicated | A single measurement campaign, a single device or a single binary; or a bytecode reading verified line by line by the investigators without a behavioural A/B. |
| **G3** | Inferred | Consistent with all measured data but not directly tested. |
| **G4** | Reported, not verified | Originates from a third party or from a pre-2026-07-08 investigation session not conducted under the evidence contract. Carries no evidentiary weight; recorded for transparency. |
| **R** | Refuted | Tested and rejected. Includes hypotheses advanced by the investigators. |

**Methodological reset.** On 2026-07-08 the investigation was restarted under an explicit "zero prior belief" contract. All claims made before that date — including measurements — were demoted to G4 and were readmitted only after being re-established by an evidence bundle produced under the contract. Where a pre-reset measurement is cited below at grade G2, it is because its result was subsequently corroborated in direction by post-reset evidence, although the original artefacts were not re-run.

## 5. Reported symptom

| ID | Content | Grade |
|---|---|---|
| S-1 | On a content-dense Compose screen, beginning a drag immediately upon touch produces a visible freeze of several hundred milliseconds before scrolling commences; touching, pausing, then dragging is smoother. | G4 (third-party report) → **G1** (reproduced in Application A on the Test device: 783 ms command-to-scroll delay, of which 348–378 ms inside the Agent's `calculateTouchInformation`) |
| S-2 | The freeze occurs on some physical devices, including high-end devices, and "never on the emulator". | **G4.** No reporter-owned device was measured. The absolute form of "never on the emulator" is refuted (H-J). |

## 6. Findings

### 6.1 Causal attribution

**F-01 — The scroll-onset stall is caused by the Agent's Jetpack Compose user-interaction instrumentation, not by the application's rendering nor by Compose itself.** — **G1**

Evidence (on/off control reproduced in three independent environments):
1. *Application A, Test device, real finger gesture (scrcpy + xdotool).* Neutralising the single woven Hook line in `NodeCoordinator.smali` reduces command-to-scroll delay from ~783 ms to ~60 ms; a build with the Agent removed was observed smooth. (Pre-reset, 2026-07-04/05; G2 in isolation.)
2. *Reproducer, Test emulator and Test device, 2026-07-09.* Agent on: main-thread gesture CPU 262 ms (emulator) / 343 ms (device), DTSTALL 269 / 342 ms, `dyn=true`, culprit frames `TouchUserInteractionHandler.generatePath` / `calculateLayoutPosition`. Agent removed: no stall, gesture CPU ~38 ms. Three instruments concurred (DTSTALL, schedstat, stack attribution).
3. *Reproducer, Test device, real gesture, `dumpsys gfxinfo`, 2026-07-13/14.* Agent on: 70–86 % janky frames, 99th-percentile frame 3.85–4.25 s, peak DTSTALL 5.12 s `dyn=true`. Agent removed: 9–19 % janky, 99th percentile 48–69 ms. GPU time ≤ 15 ms in all conditions.
4. *Application B, Agent woven, Test emulator, 2026-07-16.* See F-04.

Limitation: the multi-second figures in item 3 belong to a deliberately over-provisioned Reproducer; the Application A figure is ~0.8 s.

**F-02 — The stall is CPU-bound computation on the main thread, not idle waiting.** — **G1**

Evidence: schedstat main-thread CPU 1–5 ms during idle windows versus 262–452 ms during the gesture window, approximately equal to the DTSTALL duration, in every Agent-on condition across the Reproducer and Application B. An earlier reading ("main thread in `do_epoll_wait`, ~0 CPU", 2026-07-04) was taken on builds contaminated by the out-of-scope defect and was withdrawn (Section 8, C-3).

**F-03 — The behaviour is specific to the Agent's Compose instrumentation; the Agent's classic View instrumentation performs no comparable per-touch work.** — **G1 (Reproducer)**

Evidence: an identical structure (24 overlapping full-screen, 16-deep nested clickable layers plus a scrollable list) rebuilt with classic Android Views, Agent on: 22 ms gesture CPU, zero stalls, versus ~262 ms and a 269 ms `dyn=true` stall for the Compose equivalent. Scroll advancement verified (row 0 → 46–48). APK SHA-256 `3368a61a…e45ffc`.

**F-04 — The mechanism is not specific to Application A's content; an independently developed Compose application exhibits it under native rendering and does not exhibit it under WebView rendering of the same content.** — **G1**

Evidence (2026-07-16): one Agent-woven Application B APK (SHA-256 `fe52d206…70bf38`; probe variant `16286f92…4cb9`), same session, same content; the only variable was the render engine.

| Render engine | N per pass | `calc_ms` per pass | Main-thread CPU per swipe |
|---|---|---|---|
| WebView | 13 | 3–6 ms | 24–32 ms |
| Native Compose, mid-screen | 59 | 90–101 ms | 107–111 ms |
| Native Compose, dense card region | 35 → 61 → 139 → 139 | 42 → 135 → 276 → 275 ms | 96 → 144 → 271 → 268 ms |

Pre-registered criterion: N ≥ 60 with an Agent-attributable stall → hypothesis sustained. Observed N up to 139 exceeds Application A's measured 89–118.

### 6.2 Mechanism

**F-05 — The Agent's Gradle plugin weaves an unconditional call into the Compose framework's hit test.** — **G1**

Evidence:
- Application A decompilation, `androidx/compose/ui/node/NodeCoordinator.java` (hit method, lines 498–527): walks to the root, builds a `[semanticsId, siblingIndex]` list for the entire subtree, and calls `UserInteractionsApi.onNodeCoordinatorHit(layoutNode, hasHit, list)` with no configuration check. (Line-by-line verified reading; G2 in isolation.)
- Independent bytecode confirmation (2026-07-15): two Reproducer APKs with byte-identical Kotlin, differing solely by a Gradle `exclude` block, built with `--no-build-cache --rerun-tasks` with the weave task verified to have executed. Without the exclude: two call sites of `onNodeCoordinatorHit` in the DEX (one inside `NodeCoordinator`). With `exclude { packages("androidx.compose.ui.node") }`: one call site (the API's internal plumbing only); the ~176-line woven block is absent from `NodeCoordinator.hit`. Only `classes5.dex` differs (340 bytes). SHA-256 `a7363697…79a5f` (A) and `ecc1f63f…e7c08` (B).

**F-06 — Per pass, the Agent enumerates every attached Compose layout node whose window bounds intersect the *full bounds of the hit layout* — not the touch point — and synchronously describes each of them on the main thread.** — **G1 for the rectangle semantics and proportionality to N; G2 for the internal cost distribution**

Evidence:
- Smali of `TouchUserInteractionHandler.onNodeCoordinatorHit$`: the rectangle passed to `getOverlappedLayouts` is `Rect(hitLayout.positionInWindow, hitLayout.size)`; `getOverlappedLayouts` filters `attachedLayouts.values()` (all `LayoutInfo` registered across the composition via a woven `AndroidComposeView` attach hook) by `boundsInWindow.overlaps(rect)`; the sole guard is `isValid(layoutInfo) && !hasHit`.
- Empirical confirmation (2026-07-16): a reflection dump of the live `LayoutNode` tree of Application B at the peak scroll state counted 474 placed nodes, 191 intersecting the screen, and **exactly 139** intersecting the scroll container's content column — reproducing the N = 139 reported by the in-Agent probe. When the hit test resolves to a full-width scroll container, N ≈ every visible layout node.
- Call chain (Application A decompilation, `TouchUserInteractionHandler.java`): `calculateTouchInformation` (156–167) creates one `ComposeElementInfo` for the hit and one per overlapped layout; each invokes `generatePath` (97–102, recursive to the root) which at each level performs `calculateLayoutPosition` (45–61, linear scan of the whole-tree list built in F-05) and `generateClazz`/`getFunctionName` (63–80, 104–133, modifier-chain scan with four string post-processors).
- Asymptotic reading: O(N) enumeration plus O(O × D × P) path generation, where O = overlapped nodes, D = depth, P = size of the whole-tree list ≈ N; approximately cubic in tree size when D and O scale with N. (Analytical; measurements confirm monotonic proportionality to N, not the exponent.)
- Cost distribution: in Application A, stubbing `generatePath` to return `""` reduced `calculateTouchInformation` from 348–378 ms to 74–97 ms (2026-07-05, one device); skipping `calculateTouchInformation` entirely gave a floor of 46–65 ms (2026-07-07). `simpleperf` on the Test emulator attributed 74.1 % of sampled CPU during touch to `onNodeCoordinatorHit` (`generatePath` 61.2 %, `calculateLayoutPosition` 22.0 %, `getOverlappedLayouts` 2.4 %). On the physical device `simpleperf` does not resolve Agent symbols (interpreted code); attribution there rests on the DTSTALL watchdog's `getStackTrace`.

**F-07 — Per-pass cost is a monotonic, approximately linear function of N; N is the causal lever.** — **G1**

| Experiment (2026-07-16 unless stated) | Manipulated factor | Result |
|---|---|---|
| Within-native layer ablation in Application B, pixel-identical: four renderer-level wrapper collapses; screenshots diffed with clock/timestamp masked → zero differing pixels | Layout-node count only | N 139 → 125 (−10.1 %), `calc_ms` 269.5 → 225.3 (−16.4 %), main-thread CPU −15.8 %. Ablated (N, `calc_ms`) points interleave with baseline points on one monotonic curve (least squares over pooled x86 points: `cost ≈ 1.94·N − 6.8 ms`). |
| Pure-content null test in the Reproducer: 600 visible rows, no `alpha(0)`, no same-bounds wrappers (census: 1 root-plumbing pair vs 413 in the canonical arm), N = 134 | Composition (hidden vs visible nodes) | `dyn=true` stall reproduces (85–88 ms, 3 of 4 repetitions; `isClickable` on the stalled stack in repetition 3). Scan-isolated cost (Agent-on minus hook-removed control) 40–50 ms per swipe. Per-node price is composition-dependent (0.30–0.37 ms/node plain rows; 0.50–0.55 clickable stacks; ~1.9 in the modifier-heavy Application B) — composition modulates the constant, not the mechanism. |
| Reproducer breadth/depth series (2026-07-09) | Overlap count vs depth | Few layers ~77 ms, no stall; depth-only 16/28 ~92/131 ms, no stall; 24 layers × 16 deep ~355 ms, stall. |
| Reproducer inert variant (2026-07-09) | Actionability/semantics of nodes | Equal or worse (~310 ms, 411–439 ms stall). Cost follows overlap, not actionable nodes. |

**F-08 — For scroll/drag gestures, the Agent discards the result of the computation.** — **G2 (static analysis, verified by the investigators; no dynamic confirmation)**

Evidence (Application A decompilation, `com.dynatrace.agent.userinteraction.handler`): `FingerInteractionCoordinator.emit(SequenceState.Pending)` — the `Touch` branch (line 62) forwards `getComposeEvent()` to the touch handler; the `Gesture` branch (line 67) calls `gestureHandler.emit(interaction, durationMs, downTimestamp)` without it; `GestureUserInteractionHandler.emit` (line 55) has no such parameter. The `composeEvent` field of `Pending.Gesture` is populated (line 103) and never read. On the tap path, dispatch is gated downstream of the computation by `isTouchEnabled()`, `ServerConfiguration.isTouchUserInteractionEnabled()` (`userInteractionsCapture.contains("all")`, default empty until server sync), `isGrailEventsCanBeCaptured`, session and event-reporting checks. The sensor is instantiated unconditionally at Agent start (`UserInteractionModule.provideUserInteractionManager`, `CoreComponent`). Consequently no runtime data-collection setting avoids the cost; it can only discard the already-computed event.

Limitation: beacon interception was not performed; `DataCollectionLevel.OFF` was not exercised at runtime.

**F-09 — Frequency amplifier: a real finger gesture triggers 7–8 full passes per gesture; `adb input swipe` triggers one.** — **G2**

Evidence: a `DTPROBE` entry counter in Application A on the Test device (2026-07-07). The pass count remained 7 while the number of synthetic move samples was varied from 1 to 16, indicating that passes track re-hit-testing on layout change during scroll (`AndroidComposeView.resendMotionEventOnLayout`) rather than digitiser sample density. This explains why `adb`-driven emulators under-reproduce on Application A. Full reconciliation of device specificity remains open (O-2).

**F-10 — Agent configuration observed in Application A.** — **G2 (static analysis)**

`AgentMode.SAAS` against a live tenant; the host application applies `DataCollectionLevel.USER_BEHAVIOR` with crash reporting and crash replay opted in, at process start and on every main-activity recreation. `instrumentationFlavor` auto-upgrades to `JETPACK_COMPOSE` when a Compose dependency is present; `UserInteractionConfigImpl.isTouchEnabled()` returns true for both flavours, so the flavour is not what enables the sensor. Whether the tenant consumes Compose user-action data is not determinable from the APK.

### 6.3 Mitigations exercised

| ID | Mitigation | Result | Grade | Limitation |
|---|---|---|---|---|
| M-1 | Gradle DSL `exclude { packages("androidx.compose.ui.node") }` | Removes the woven call site (F-05); zero DTSTALL in 4 of 4 repetitions versus 439–445 ms `dyn=true` in 3 steady repetitions; steady gesture CPU 10–40 ms versus 286–452 ms. Also confirmed as the hook-removed control in the null test (F-07). | **G1 in the Reproducer; G3 for Application A** | Originated in a third-party report and was corroborated, not discovered, by the investigators. Application A was not rebuilt (source unavailable). Removes the Agent's Compose touch→element attribution. Valid for plugin 8.341.1.1004; hook location must be re-verified for other versions. |
| M-2 | Dynatrace plugin ≤ 8.269.1.1009 | 8.269: one 173 ms first-touch warm-up, then zero stalls ≥ 80 ms in an eight-swipe burst; 8.341: 14 stalls of 362–4 337 ms, all `dyn=true`. The 8.269 APK contains no `com.dynatrace.agent.compose.userinteraction` package (only the legacy `com.dynatrace.android.compose.*` wrappers). | **G1 in the Reproducer; G3 for Application A** | Compose auto-instrumentation became default-on in 8.271. 8.343.1.1038 remains affected (Application B). Measured on a second emulator host, Android 15. |
| M-3 | Rendering in a WebView or in classic Android Views | WebView: N = 13, no stall (F-04). Views: 22 ms, no stall (F-03). | **G1** | |
| M-4 | `userActions { composeEnabled(false) }` | Woven Hook in `NodeCoordinator` byte-identical; stall persists (458–499 ms). Clean rebuild without cache. In the Kotlin DSL the setting compiles only as a method call (the field is private). | **R** | See C-1. The flag disables a different subsystem (click/toggle callbacks), not the hit-test hook. |
| M-5 | `exclude { packages("<application package>") }` | Stall persists and worsens ~1.8×. | **R as mitigation; G1 as measurement** | Not to be confused with M-1: removing the application's type hints makes the unconditional walk more expensive. |
| M-6 | Flattening wrapper layers of the UI without visual change | −10 % N → −16 % cost (F-07). | **G1 as measurement; insufficient as mitigation** | Reaching imperceptibility would require collapsing whole content subtrees into draw nodes. |
| M-7 | Reducing actionable/semantic nodes | No reduction. | **R** | |
| M-8 | Memoising or de-duplicating the ~4 scans per gesture; spatial index for `getOverlappedLayouts` | No benefit: each pass occurs at a different scroll offset; the expensive step is description, not lookup (`getOverlappedLayouts` 9–10 ms). | **R** (Application A, one device — G2 measurement) | |
| M-9 | Temporal debounce of `calculateTouchInformation` to ~1 pass per gesture (Agent-side) | Proposed on the basis of F-09 (1 pass does not stall, 7–8 do). | **G3** | Not implementable by the application. |
| M-10 | Drawing static content into few `Canvas` nodes to reduce N | Prototype collapsed node count by construction but produced a monolithic canvas that was heavy and did not scroll; not pursued. | Not concluded | On-device N measurement with the Agent was never performed for this prototype. |
| M-11 | Hosting the Compose UI in a secondary process | Analysis and proof of concept only. | **G3** | OneAgent ≥ 8.327 starts only in the default process. |
| M-12 | Application-side render-cost throttling | A/B under CPU throttle: ANR unchanged; regression of 36 %. | **R** | |
| M-13 | Replacing Material3 `1.10.0-alpha05` with stable `1.9.0` | No improvement (99th percentile 650–700 vs 700–1 100 ms, both with Agent). | **R** for the hypothesis that the alpha dependency is causal | The instrumented hit test resides in `compose-foundation` 1.10.0 (stable). |

## 7. Register of rejected hypotheses

| ID | Hypothesis | Method | Verdict |
|---|---|---|---|
| H-A | A specific heavy component (carousel/chart) is the cost | Removal and re-measurement | R — delay essentially unchanged |
| H-B | CPU/RAM/GC pressure or UI-library callbacks | schedstat idle vs gesture | R — computation during gesture, not waiting or GC |
| H-C | The hit-test BFS is the hotspot | Section timing | R — ~0 ms |
| H-D | `getOverlappedLayouts` alone explains the cost | Timing | R — 9–10 ms; the loop over its result is the cost |
| H-E | Off-centre touch (48 dp near-candidate) | Centre vs gap vs empty A/B on a grid | R — mechanism inverted (centre expensive, gap cheap) |
| H-F | Tree depth alone | Depth-only 16/28 with Agent on | R — no stall |
| H-G | Cost scales with actionable nodes | Inert variant | R — scales with overlap |
| H-H | The mechanism requires hidden layers (investigators' hypothesis) | Pre-registered null test, N = 134 visible content | R — reproduces with zero hidden layers |
| H-I | Material3 alpha is causal | Downgrade to stable | R |
| H-J | "Never on the emulator" is absolute | Over-provisioned Reproducer on x86_64 | R as absolute; does **not** establish that Application A stalls on an emulator |
| H-K | Main thread is waiting, not computing | schedstat | R |
| H-L | The out-of-scope scroll-steal defect explains this stall | Before/after its correction | R — distinct defect |
| H-M | `composeEnabled(false)` removes the instrumentation (investigators' hypothesis) | Clean rebuild, byte-level re-decode, runtime | R |
| H-N | Application-package `exclude` removes the instrumentation | Single-variable A/B | R — ~1.8× worse |
| H-O | A stall in Application B falsifies the overlap hypothesis ("it has no overlays") | N probe in Application B | R — premise false: N = 59–139 under native rendering |
| H-P | Lowering refresh rate (120 → 60 Hz) or forcing 30 fps resolves | Reasoning and pass-count measurement | R — cost per pass, not touch rate, is the driver |
| H-Q | Application-side render throttling resolves | A/B under throttle | R |

## 8. Corrections and retractions

Statements made by the investigators during the investigation and subsequently found to be incorrect.

| ID | Statement retracted | Asserted | Corrected | Basis |
|---|---|---|---|---|
| C-1 | "`userActions { composeEnabled(false) }` is the only clean fix, verified by reverse engineering." | 2026-07-04/05 | 2026-07-13/14 | Clean rebuild of the Reproducer: Hook byte-identical, stall persists. Error cause: inferring build-plugin behaviour from runtime Agent bytecode. |
| C-2 | "No configuration-based mitigation is known." (Interim report of 2026-07-14.) | 2026-07-14 | 2026-07-15 | Over-broad: only the application-package exclude had been tested. The framework-package exclude (M-1) was corroborated. **The 2026-07-14 interim report is superseded on this point.** |
| C-3 | "The main thread is idle-waiting (`do_epoll_wait`, ~0 CPU)." | 2026-07-04 | 2026-07-04/09 | Measurement contaminated by builds with the out-of-scope defect or a WebView fallback; schedstat on clean builds shows computation. |
| C-4 | "The stall never reproduces on an emulator." | 2026-07-05 | 2026-07-09 | Reproduced on x86_64 with an over-provisioned tree. |
| C-5 | "Device specificity is explained by off-centre touch landing." | 2026-07-08 | 2026-07-09 | Grid A/B inverted the predicted pattern. |
| C-6 | An artefact labelled as the `composeEnabled(false)` build was in fact built without the flag. | 2026-07-09 | 2026-07-13 | Detected by bytecode comparison; replaced by clean rebuild pairs. No conclusion depended on the mislabelled artefact. |
| C-7 | "The mechanism requires hidden (invisible/redundant) layers." (Investigators' position in a dispute with a colleague.) | 2026-07-16 | 2026-07-16 | Pre-registered null test refuted it; the count-driven model was upheld, with the refinement that the per-node constant is composition-dependent. |
| C-8 | "A gauge component is composed of many individually laid-out tick nodes." | 2026-07-16 | 2026-07-16 | UI-library source: the arc is drawn inside a single `Canvas` node. The N = 139 is diffuse (50 Column, 39 Box, 26 Text …), not one component. |
| C-9 | "The ~300 Hz digitiser sampling rate is the frequency amplifier." | 2026-07-06/07 | 2026-07-07 | Pass count (7) invariant to the number of move samples (1–16); passes track layout-driven re-hit-testing. |

## 9. Open items and limitations

| ID | Item | Status | Bearing on the conclusions |
|---|---|---|---|
| O-1 | A single physical device was measured. Reporter-owned devices were not measured. | Open | Does not affect component-level attribution (the on/off control is device-independent); affects the estimate of magnitude per device. |
| O-2 | Complete reconciliation of device specificity. Indications: real gesture = 7–8 passes vs `adb` = 1; arm64 ≈ 4× slower per node than x86. | Open (G3) | Low. |
| O-3 | Application A was never exercised on an emulator (authenticated session required; backend unavailable during the investigation). | Not tested | The emulator Reproducer is over-provisioned; it shows the emulator is not immune, not that Application A stalls there. |
| O-4 | M-1 and M-2 were not applied to Application A (source unavailable). | Not tested (G3) | Mitigations demonstrated only in the Reproducer and Application B. |
| O-5 | Plugin configuration space: only `composeEnabled`, two `exclude` targets and three plugin versions (8.269, 8.341, 8.343) were exercised. Granular flags (`composeSemantics`, `composeClickable`, …) are consumed at build time and were not tested individually. | Partial | Another setting may also remove the Hook. |
| O-6 | Agent versions newer than 8.343.1.1038 were not tested. | Not tested | The vendor may have addressed the behaviour. |
| O-7 | Whether the tenant consumes Compose user-action data. | Not determinable by the investigators | Determines the practical cost of M-1/M-2. |
| O-8 | Dynamic confirmation of F-08 (beacon interception). | Not performed | F-08 remains G2. |
| O-9 | Exact per-sub-function cost distribution within the hotspot. | G2 | Component-level attribution unaffected. |
| O-10 | Third-party statements about remediation status of Application A. | G4 | Unverified single-source reports. |
| O-11 | Side effects of M-1 on the Agent's other capabilities (crash, ANR, network, app-start). | G3 (expected intact: the exclude removes only the framework call site; Agent classes remain) | Not measured. |
| O-12 | Instrument limitations: a scroll-callback-gap metric proved a false negative for in-scroll jank and was replaced by gfxinfo/DTSTALL; `simpleperf` does not resolve Agent symbols on the physical device; `dumpsys gfxinfo` is indirect and used only as a secondary metric. | Known | Accounted for in grading. |
| O-13 | Not all pre-2026-07-08 raw logcat/dump artefacts were preserved; some results exist only as contemporaneous consolidations. The probe↔patched bytecode diff of Application A is not documented separately. | Known | Pre-reset material is graded G2 or G4 accordingly. |

## 10. Chronology

| Date | Event |
|---|---|
| 2026-07-03 | Decompilation of Application A begins. |
| 2026-07-04 | Out-of-scope scroll-steal defect separated and corrected. First attribution to the Agent's Compose Hook by reverse engineering. "Waiting" reading identified as contaminated. |
| 2026-07-05 | Hotspot measured in Application A on the Test device (348–378 ms of 783 ms); `generatePath` A/B; embedded Agent configuration extracted; Agent-removed control built. Canvas prototype (M-10) trialled and set aside. |
| 2026-07-06 | Topology survey; secondary-process analysis (M-11). |
| 2026-07-07 | Skipping `calculateTouchInformation` → 46–65 ms floor. Pass count: real finger 7–8 vs `adb` 1. |
| 2026-07-08 | Methodological reset; evidence repository created; all prior claims demoted to G4. Reproducer built with Application A's exact dependency versions (Kotlin 2.3.0, AGP 8.11.2, Compose MP 1.10.0, Material3 1.10.0-alpha05, plugin 8.341.1.1004); weaving confirmed in the ASM log. |
| 2026-07-09 | First emulator reproduction, triangulated by three instruments; Views control (22 ms vs 262 ms); off-centre hypothesis refuted; application-package exclude refuted. |
| 2026-07-13/14 | Perfetto and `simpleperf` captures in both environments; Reproducer on the physical device (stalls to 5.1 s); `composeEnabled(false)` tested and refuted. Interim report issued. |
| 2026-07-15 | Framework-package exclude (third-party report) corroborated by bytecode diff and 4-repetition A/B. Version boundary established (8.269 unaffected, 8.341 affected). |
| 2026-07-16 | Application B: WebView N = 13 vs native N = 59–139 in one APK. Pixel-identical ablation: −10 % N → −16 % cost on the same curve. Pure-content null test: hidden layers not necessary. **Causal closure.** |

## 11. Method summary

- **Environments.** Test device accessed through a remote build host (adb loopback-only). Test emulator on the same host. A second emulator host for M-2.
- **Gesture generation.** `adb input swipe` (one pass; under-reproduces on Application A; adequate for the over-provisioned Reproducer) and scrcpy + xdotool (real gesture, 7–8 passes; used for Application A and gfxinfo measurements).
- **Instruments (triangulation required for any stall claim).** DTSTALL; schedstat; DT_PROBE; `dumpsys gfxinfo` (secondary); Perfetto; `simpleperf`; reflection dump of the live `LayoutNode` tree for an Agent-independent census of N.
- **Controls.** Every reported Agent-on figure has an Agent-off or Hook-removed counterpart from the same source, same day, same protocol. The first gesture after launch is discarded as warm-up. Builds where a byte-identical outcome was admissible were produced with `--no-build-cache --rerun-tasks` and execution of the weave task was verified. Where the investigators and a colleague disagreed (F-04, F-07), verdict criteria were written before data collection and are quoted verbatim in the evidence bundles.
- **Rigour guards.** All APKs are referenced by SHA-256; a pre-flight script refuses execution against an unexpected APK hash or device identity. Readings taken during a documented host-contention incident (2026-07-16, before ~07:20 UTC) were discarded in full.

## 12. Evidence inventory

Artefacts are retained by the investigators and are available to the auditor on request under confidentiality. Locations and application-identifying material are withheld from this document.

| Bundle | Content |
|---|---|
| E-01 | Hypothesis register (pre-reset testimony converted to falsifiable hypotheses; provenance-tagged). |
| E-02 (2026-07-09) | Reproducer: emulator reproduction (FINDING), mitigation log (inert, Views), off-centre refutation, application-package exclude refutation; `repro.sh`, `result.json`. |
| E-03 (2026-07-15) | Framework-package exclude corroboration: bytecode diff, call-site census, 4-repetition A/B; two SHA-pinned APKs. |
| E-04 (2026-07-16) | Application B overlap-count study (WebView vs native), pixel-identical layer ablation, pure-content null test; SHA-pinned APKs, census dumps, screenshots, `MANIFEST.sha256`. |
| E-05 | Reproducer source, self-contained (dummy beacon); ≥ 12 ablation APKs; measurement harnesses; Perfetto and `simpleperf` traces. |
| E-06 | Application A ablation builds (probe / `generatePath`-stubbed), measurement scripts, `MANIFEST.sha256`. **Third-party proprietary content; not for redistribution.** |
| E-07 | Application A decompilation (basis of all `file:line` references). Third-party proprietary. |
| E-08 | Version-boundary APK pair (plugin 8.269 vs 8.341). |
| E-09 | Interim report of 2026-07-14 with Annexes A–D (code trace, measurements, chronology, discards). Superseded on the point recorded in C-2. |
| E-10 | Pre-reset session reports (G4 unless re-established). |

## 13. Statement of conclusions

1. **Demonstrated (G1).** The scroll-onset stall is produced by the Dynatrace OneAgent 8.341.1.1004 Compose user-interaction sensor, woven at build time into `androidx.compose.ui.node.NodeCoordinator.hit`; its per-pass cost is proportional to the number of Compose layout nodes intersecting the full bounds of the hit layout and is incurred synchronously on the main thread; it is specific to the Agent's Compose instrumentation; removing the woven call site (framework-package exclude) or using plugin ≤ 8.269 eliminates the stall in the Reproducer; WebView and classic View rendering are unaffected; an independently developed Compose application reaches a node count sufficient to trigger the behaviour.
2. **Measured with a single line of support (G2).** Sub-function cost distribution; pass count per gesture; discarding of the computed result on the gesture path; the Agent configuration observed in Application A.
3. **Inferred (G3).** That M-1 and M-2 are effective in Application A; the complete account of device specificity; the absence of side effects of M-1.
4. **Reported, unverified (G4).** Occurrence on the reporter's high-end devices; non-occurrence on emulators for Application A; the remediation status of Application A.
5. **Superseded prior statement.** The 2026-07-14 interim report's assertion that no configuration-based mitigation exists is withdrawn (C-2); the framework-package exclude is a demonstrated mitigation in the Reproducer.

---

*End of document.*
