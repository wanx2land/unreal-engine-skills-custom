---
name: ue-motion-matching
description: "Use this skill when setting up or debugging Motion Matching (PoseSearch plugin) in Unreal Engine 5.6+: Pose Search Database (PSD), schema (PSS), Chooser-driven database selection, trajectory generation (CMC or Mover), Pose History node, or symptoms like wrong-direction clip selection, transitions that never happen (stuck idle/walk/crouch/jump), or unstable clip flip-flopping. Complements ue-animation-system (general AnimGraph) and ue-character-movement."
metadata:
  version: 1.0.0
---

# UE Motion Matching (PoseSearch)

You are an expert in UE 5.6+ Motion Matching built on the PoseSearch plugin.
This skill encodes hard-won pitfalls from real debugging sessions. Every rule
below was violated once and cost hours; read the Pitfalls before touching
assets.

## Context Check

1. Movement component: CharacterMovementComponent (CMC) or Mover 2.0? This
   decides the trajectory generation path and one critical facing fix (P1).
2. Mesh convention: does the skeletal mesh component have the standard
   mannequin relative yaw of -90 degrees? (Actor forward +X, asset forward +Y)
3. Single Motion Matching node with switched databases (GASP
   ExperimentalStateMachineData style) or one big database?
4. Do animation clips have real root motion in their root bone track?
   (`bEnableRootMotion=true` + `bForceRootLock=true` is CORRECT for
   capsule-driven characters: indexing reads the raw root track, runtime
   playback keeps the root locked.)

## Architecture that works (multi-database, state-driven)

```
C++ state machine (thread-safe, edge detection + latch timers)
  └─ LocomotionState enum → Chooser table → UPoseSearchDatabase
ABP BlueprintThreadSafeUpdateAnimation
  └─ SelectedDatabase = EvaluateChooser(...)
MM node "On Update" anim node function (C++ UFUNCTION, BlueprintThreadSafe)
  └─ SetDatabaseToSearch(SelectedDatabase, conditional InterruptMode)
AnimGraph: [Motion Matching] → [Pose History (trajectory pin)] → [Output]
```

Key decisions, each mandatory (see Pitfalls for what happens otherwise):

- **Do NOT bind the MM node's Database pin directly.** Pin binding cannot
  interrupt the continuing pose, so "pose-only" transitions (stance, jump)
  never win the search. Use `UMotionMatchingAnimNodeLibrary::SetDatabaseToSearch`
  from an On Update anim node function.
- **Interrupt mode must be conditional.** Request `InterruptOnDatabaseChange`
  only on edge frames where trajectory stays the same but pose must change
  (crouch flag flip, grounded/airborne flip). Use `DoNotInterrupt` otherwise
  so one-shot transition clips (IdleToRun, land) play to completion and hand
  off naturally at their end.
- **A normalization set covering ALL databases is mandatory.** "Single-DB
  search" does not exist: every search also evaluates the continuing pose
  from the previous database, so every DB switch is a cross-DB cost
  comparison. All databases must share one UPoseSearchNormalizationSet and
  cost-comparable schemas (start bring-up with ONE shared schema).
- **Implement the node-update function in native C++**, not Blueprint. The
  BP function's thread-safe flag can only be set in the editor UI; without it
  the compiler silently drops the binding (T-pose). A native
  `UFUNCTION(BlueprintCallable, meta=(BlueprintThreadSafe))` with signature
  `(const FAnimUpdateContext&, const FAnimNodeReference&)` binds reliably via
  `updateFunction = {className: None, functionName: <name>}`.

## Pitfalls (each observed in production debugging)

### P1. Mover trajectory facing is 90 degrees off (wrong-direction clips)

Symptom: forward input selects left-strafe clips; backward selects mirrored
left; debug trajectory line looks correct (it is drawn in world space, which
proves nothing about query space).

Cause: the Pose History node's Trajectory input expects samples as the
**skeletal mesh component (= root bone) world transform** history/prediction.
`UMoverTrajectoryPredictor` emits facing as the **actor rotation** (X-forward).
With the mannequin -90 mesh convention the whole query is rotated 90 degrees.
The CMC path (`PoseSearchGenerateTrajectory (for Character)`) corrects this via
ACharacter internals; the Mover predictor path does not.

Fix: keep the raw predictor output in one variable (pass it back as InOut for
history accumulation) and derive a corrected copy each frame in
`NativeThreadSafeUpdateAnimation`:
`Corrected.Facing = Raw.Facing * MeshComponent->GetRelativeTransform().GetRotation()`.
Feed the corrected trajectory to the Pose History pin. Never rotate the
persisted InOut variable in place (history would double-rotate).

### P2. Transitions that never happen = continuing pose wins by construction

With `UseContinuingPose`, the query's pose features are copied from the
currently playing clip, so its pose cost is 0 while any pose-different
candidate pays full cost. When the trajectory does not change either
(stand<->crouch, ground->air at takeoff, walking<->crouch-walking), the correct
clip in the new database can NEVER win. This is not a tuning problem; it
requires the interrupt architecture above.

### P3. Cost biases must match the measured cost scale

`ContinuingPoseCostBias` / `LoopingCostBias` values copied from samples (GASP:
-1.5/-0.5, -2.0) are meaningless without the same normalization. Read actual
costs in the Rewind Debugger (Active Pose row: Cost / Bias / Traj / Pose). If a
perfect steady-state match costs ~0.15, a -2.0 bias locks wrong picks for
seconds. Start near -0.05 for loop databases, -0.4 for one-shot transition
databases (helps them play to completion), and adjust against measured costs.
These biases are runtime-only (excluded from DDC hash) — changes apply without
reindexing.

### P4. Content with zero root motion cannot be direction-matched — lock it

If clips differ only in body pose (e.g. in-air loops with no root translation),
trajectory features carry no signal, and per-frame search oscillates between
near-equal candidates and mirror twins (visible body wobble). Do not try to fix
this with features: give that database a strong continuing bias (-2.0) so the
first pick (chosen by pose continuity from the previous clip) stays locked, and
rely on state-edge interrupts for entry/exit.

### P5. Trajectory Z features: adding and removing both have traps

- XY-only features + no interrupt architecture: airborne queries look identical
  to grounded ones, air clips rarely win.
- Adding 3D Position/Velocity: the fall arc's large Z mismatches every air clip
  (they hover near apex), drowning XY direction differences → random direction
  picks in air.
- Resolution: keep features XY-only and let the state machine + edge interrupts
  force database switching. Do not use features to discriminate what states
  already discriminate.

### P6. Transition databases usually cover only forward

Start/Stop DBs typically contain only `*_F` clips. Gate the transient states in
the C++ state machine (e.g. enter Start/Stop only when |MoveDirection| <= 45
degrees; skip StanceTransition while moving) or sideways/backward starts play a
mismatched forward clip for the whole latch duration ("one-beat delay" before
the correct direction).

### P7. Misc facts worth knowing

- `bEnableRootMotion=true` + `bForceRootLock=true` on clips is correct for
  capsule-driven MM: database indexing extracts the raw root track regardless
  of root lock; runtime playback keeps the mesh in place.
- First PIE of a session: `AsyncBuildIndex` warnings mean searches are skipped
  for a few frames (character stays in previous pose). Editor-session-start
  artifact only.
- Chooser table rows cannot be added programmatically (`resultsStructs` is not
  reflection-exposed) — row edits are editor-UI-only work.
- If both a full-body mesh and a camera-attached first-person mesh run the same
  MM AnimBlueprint, the FP mesh's query inherits camera pitch. Treat FP-mesh MM
  quality as a separate problem.

## Debugging workflow (do this FIRST, before hypothesizing)

1. **Rewind Debugger** (record → reproduce → inspect):
   - Blend Weights rows = which clips actually played (the single most
     valuable signal; wrong-direction and flip-flop bugs are obvious here).
   - Pose Search track: composite row labels like "DB_A - DB_B" mean the
     continuing pose came from another database that frame.
   - Details panel Active Pose: Cost / Bias / Traj / Pose breakdown — use it to
     scale biases (P3).
2. **Numeric trajectory dump**: read the trajectory variable off the live
   AnimInstance (editor Python) and verify per sample: facing yaw vs velocity
   yaw vs actor yaw vs mesh yaw. This catches P1 in one look.
3. Verify asset-space conventions once with a root-motion dump
   (`AnimPoseExtensions.get_anim_pose_at_time` at t=0/mid/end): mannequin
   forward clips move +Y with root yaw 0.
4. Only then touch schema weights or biases.
