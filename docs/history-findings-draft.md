# Project History — Working Findings Document

**Status: DRAFT / staging area. Not part of the compiled thesis. Nothing here has been
merged into the `.tex` chapters yet — this is a running log while we reconstruct the full
project history from git, changelogs, memory, status PPTs, and chat summaries, before deciding
what goes into the report and how.**

Sources consulted so far: git log (`qcar_hardware` repo), `qcar_initial`/`qcar_ackermann`/
`qcar_updated` CHANGELOG.md files, `qcar_navigation` memory, 8 status PPTs (2026-04-27 to
2026-08-17), 3 chat summaries (chat-till-June-17, a parallel qcar_ackermann chat, and this
conversation's own MPPI cusp-freeze work).

Package lineage: `qcar_initial` (baseline) → `qcar_ackermann` (throwaway test package for the
Ackermann-plugin prototype, merged back) → renamed to `qcar_updated` (final package).

---

## Phase 0 — Planning (2026-04-27)
- Scoped: ROS2 SLAM+Nav2 for QCar, originally including EKF/UKF sensor fusion (dropped later,
  never implemented — replaced by dead-reckoning odometry).
- Started on ROS 2 Foxy in Docker; not yet Humble.

## Phase 1 — ROS 2 Humble bring-up (~early May, PPT 2026-05-11)
- Upgraded to Ubuntu 22.04 + ROS 2 Humble natively.
- URDF (received, not built from scratch) adapted: transmission tags, D435 camera macro, control
  plugin fixed for ROS2 compatibility.
- First working Gazebo spawn + RViz with correct TF + `ros2_control` stack + teleop.
- **Status: not in thesis at all** — the whole foundational bring-up chapter.

## Phase 2 — First SLAM attempts, `qcar_initial` (mid-May – early June)
- Earlier `diff_drive_controller` attempt abandoned (wheel-orientation conflicts from
  `rpy="0 0 3.14"`) → settled on JointGroup controllers + `cmd_vel_to_drive.py` (= thesis's
  current "Baseline" section).
- PPT 2026-05-25: LiDAR + Cartographer added, first map built. **Issue**: "Robot orientation not
  updating in RViz" — root cause named as QoS mismatch between `joint_state_broadcaster` and
  `robot_state_publisher`.
- PPT 2026-06-08: Nav2 launched (imprecise). **Issue**: robot pose static in RViz while moving in
  Gazebo — refined root cause: missing odom→base transform, since `drive_controller` never
  published odometry.
- **Status: partially in thesis** (end-state ground-truth-odom decision is in §3.2; the TF
  debugging itself is not narrated).

## Phase 3 — `qcar_ackermann`: prototyping the Ackermann plugin (mid-June)
Built as a throwaway test package to prove dropping `ros2_control` for
`libgazebo_ros_ackermann_drive`, later merged back into `qcar_initial`. Bug chain, from the
chat-till-June-17 + parallel qcar_ackermann chat:
- Launch bugs: `robot_description` YAML parse error (`Command()` not typed as string), unreliable
  Gazebo launch pattern, spawn race condition.
- False lead: "robot spawned without wheels" — actually LiDAR ray-visualization obscuring the
  view, not a real defect.
- Dual actuation conflict — old and new drive plugins both active, fighting over `/cmd_vel`.
- Ackermann joint-mapping bug — wrong joint names, steering silently ignored.
- **LiDAR TF bug** — `continuous` joint with no `/joint_states` source, blocked all of Nav2.
  Likely the real cause of the broken-TF screenshots shown earlier in this conversation.
- Wheel-flip-on-joint-axis (different fix than the axis-negation already in the thesis for the
  ros2_control-based migration — same symptom class, different correct fix depending on consumer).
- PID gains entirely absent → zero torque → free drift (found by reading plugin source).
- Wheel vibration from concave mesh collision; fixing it unmasked missing joint damping.
- PID retuning needed because plugin demo defaults (`1500 0 1`) were tuned for a full-size car,
  not a 2.4kg QCar.
- **DDS `ROS_DOMAIN_ID` bug** — picking up another machine's ROS2 nodes over the network. Good
  concrete pairing for the DDS/SPDP discovery theory already in Ch2.
- Broken teleop script after the plugin switch.
- Wheel radius=0 bug — same defect signature as what's already in Table `tab:not-moving-bugs`
  bug #1, caught here mid-investigation.
- **Status: mostly not in thesis** — only the final wheel-radius/friction fix made it into the
  existing table.

## Phase 4 — Package rename + first Nav2/MPPI maturation (July 2026)
- 2026-07-01 to 07-05 (git): `qcar_initial` → renamed to `qcar_updated`.
- 2026-07-05 (git/CHANGELOG): migrated JointGroup controllers → `ackermann_steering_controller`
  (**= thesis's First Migration, in thesis**).
- 2026-07-13 (git/CHANGELOG): migrated → `libgazebo_ros_ackermann_drive` directly, fixed
  "not moving"/"not reaching goal" (**= thesis's Second Migration + Table tab:not-moving-bugs,
  in thesis**).

## Phase 5 — The goal-approach oscillation → planner/controller saga (2026-07-14, all one day)
**This is NOT a clean one-shot fix. It's an iterative arc that changed both the planner and the
controller, and even after 9+ rounds was not claimed as fully solved. Document it as an ongoing,
partially-resolved investigation, not a solved bug.**

Starting symptom: robot reaches goal position but swings/wobbles correcting final orientation.

1. Root cause: `NavfnPlanner` has no heading concept at all; default BT replans every second
   continuously, even after reaching goal position. Removed `<Spin>` recovery (physically
   impossible for this car) — **already in thesis, §sec:bt-fix**. *Tried and reverted*: a
   "replan only if path invalid" BT — real bug in this Nav2/Humble version
   (`IsPathValid` returns SUCCESS on an empty/uninitialized path before `ComputePathToPose` runs).
2. Slowed replan rate (1→0.2Hz) — partial help, but `allow_reversing:true` gave the controller a
   second degree of freedom to flip-flop on.
3. Real fix (for this specific symptom): `progress_checker` was aborting mid-maneuver —
   `SimpleProgressChecker` in this Humble version only judges linear displacement
   (`required_angular_distance` silently a no-op), so small heading-correction moves looked
   "stuck."
4. Workaround tried and rejected by user: loosen `yaw_goal_tolerance` to ~166°. User said heading
   precision genuinely matters — do not present this as the accepted fix.
5. **Real architectural fix**: switched global planner `NavfnPlanner` → `SmacPlannerHybrid` with
   Reeds-Shepp motion model. **This is the change that introduced K-turns/cusps into planned
   paths at all** — before this, per the user's own account (2026-08-31 message), planned paths
   were continuous, without direction reversals; an Ackermann vehicle can't always achieve every
   goal that way, so this was a necessary and correct change, but it created an entirely new
   class of problem (cusp handling) that hadn't existed before.
6. Re-applied the replan-rate slowdown now that `progress_checker` was patient.
7. **Discovered `RegulatedPurePursuitController` (RPP) cannot handle cusps at all** in this
   Nav2/Humble version — single lookahead "carrot point" goes unstable exactly at a
   direction-reversal point. Workaround: forced planner to forward-only Dubins paths (no cusps).
8. User **rejected** the forward-only workaround — reversing is a real requirement. Re-enabled
   reversing with much more conservative RPP tuning (tighter lookahead, slower approach to tight
   curvature).
9. **Still failed intermittently** even after (8)'s tuning — one goal 85s, a near-identical one
   7.6 minutes before giving up. Concluded this is RPP's inherent algorithmic limitation, not
   tunable. **Switched local controller to MPPI.**

### MPPI did not immediately solve it either — more rounds followed, same day/next day
10. `enforce_path_inversion` was missing entirely — MPPI had no idea how to handle a cusp/reversal
    path at all after the switch.
11. `CostCritic.consider_footprint` silently defaulting to `false` — collision checked at robot
    center only, causing corner-cutting. Fixed, and replaced the costmap's circular
    `robot_radius: 0.15` approximation with a real rectangular footprint measured off the STL
    (`0.44m × 0.18m` — same number already used in the thesis's near-wall-tolerance section).
12. **First live-verified test in this whole thread** (previously relying on user-reported logs):
    a goal succeeded but not cleanly (0.202m/6.97° error); a second, harder goal **timed out
    completely** — same RPP-style failure signature reproduced even under MPPI. Isolated via live
    A/B param testing that `consider_footprint: true` (the correctness fix in (11)) is itself a
    real, understood trade-off — an accurate footprint is harder to keep collision-free in a tight
    K-turn than the old undersized circle was.
13. AMCL jitter tuning — measured live `map→odom` TF jumps up to 7.2cm/2.3°. Tuning helped the
    *character* of the jitter (more frequent, smaller corrections) but did **not** fix the
    underlying goal-reaching failures.
14. **A clean isolation test ruling out vehicle physics**: bypassed all of Nav2, published raw
    `/cmd_vel` directly, measured TF. Pure reverse: 2mm lateral drift. Reverse+steering together:
    smooth, monotonic, no instability. Proved the vehicle itself can execute a clean K-turn — the
    remaining failure was entirely in the Nav2 stack, not the robot.
15. Root cause found: `PreferForwardCritic` (penalizes reverse) vs. `PathFollowCritic` (stops
    enforcing the plan inside 1.4m of goal) left a 0.5m–1.4m "dead zone" where nothing enforced
    the plan but reverse motion was still actively penalized — exactly where a K-turn's reversal
    segment falls. Removed `PreferForwardCritic`, closed the dead zone. A previously
    catastrophically-failing goal (2.2m/177° error) converged to 0.145m/0.22°.
    **Still not presented as a complete fix even in the source material** — more issues followed.

### Further issues specifically caused by the new K-turn/cusp paths (2026-07-18/19)
16. Four stacked bugs behind a goal still not succeeding:
    - A BT.CPP `Fallback`/`SingleTrigger` interaction silently re-armed a one-shot trigger,
      causing **33Hz continuous replanning**. Fixed by using a plain `<Sequence>` instead.
    - **Planner and controller shared the exact same minimum turning radius (0.5m) — zero
      margin.** Any MPPI sample noisier than perfect got penalized, so MPPI preferred staying
      still. Fixed by giving the planner more slack (0.7m) than the controller's true hard limit
      (0.5m). **This is the "increase the turning radius" step the user specifically
      recalled.**
    - MPPI's own warm-start (sampling noise around the *previous* cycle's control sequence) made
      a stall self-reinforcing once it collapsed toward zero velocity.
17. `EarlyCommitCritic` (built to fix a separate "robot doesn't turn at the start" problem) was
    found fighting legitimate reverse/K-turn segments — it assumed forward travel only, scoring a
    correct reverse maneuver as maximally wrong. Fixed with a direction-aware
    `min(fwd, reverse)` bearing distance. **Already reflected in the thesis's existing
    `EarlyCommitCritic` description in Ch6.**
18. 2026-07-20: `minimum_turning_radius` raised further, 0.7→1.0, paired with `yaw_goal_tolerance`
    0.3→0.5 — the last user-tuned accuracy fix visible in this cluster.
19. **2026-08-02, full detail (read directly from changelog)**: corner-cutting/path-deviation
    rigorously investigated on the user's own reported goal. Notably, **the investigation's first
    pass had its own bug** — comparing live position against a stale `/plan` snapshot instead of
    whichever plan was actually active at each timestamp, producing meaningless numbers; fixed by
    capturing every distinct plan snapshot with its own timestamp. With that fixed, measured a
    real, repeatable peak deviation of 0.44m at 60-70% through the maneuver, converging back to
    ~0.11-0.13m near the goal. Five parameters tested with an actual numeric harness (not
    impression): steering PID `kd`, `PathFollowCritic`/`PathAngleCritic.offset_from_furthest`
    (at two different values), `PathAlignCritic.cost_weight` (lowered this time, having only been
    raised before), `GoalAngleCritic.threshold_to_consider`, and `wz_std` alone — **none produced
    a change clearly beyond single-trial noise**. Synthesis: contrasted against a milder goal
    showing much smaller deviation, concluded the goal's own geometry (~180° heading change over
    ~2m net translation, close to the vehicle's kinematic limits) is the dominant driver, not any
    single tunable parameter — a genuine, evidence-based "this is a goal-shape-dependent
    limitation" conclusion, not an unexamined shrug.

### Honest framing for the thesis (per user's correction, 2026-08-31)
This entire arc should NOT be written up as "problem → root cause → fixed." It should be written
as: switching to Reeds-Shepp path planning was necessary and correct (an Ackermann vehicle cannot
always reach a goal via a continuous, non-reversing path), but it introduced K-turns/cusps as an
entirely new class of problem that the existing controller (RPP) couldn't handle at all, motivating
the MPPI switch — which itself then needed multiple further rounds of fixes specifically because of
the new cusp-containing paths, and even after all of them, the user's own assessment is **"close to
perfect" but explicitly not a 100% fix**. This is consistent with — and a much richer version of —
the MPPI reversal-cusp-freeze work already in Ch6 (§sec:cusp-freeze), which picks up this same
thread later (2026-08-25 onward) with the dedicated `CuspStraightenerSmoother`. These two
investigations are the same underlying problem (K-turn/cusp handling) at two different points in
the project's timeline, and should probably be connected explicitly rather than presented as
unrelated.

---

## Phase 5b — More from `TUNING.md` (the project's own living cross-reference, checked 2026-08-31)
`TUNING.md` is a consolidated, cross-referenced summary the project kept up to date throughout —
it independently confirms everything in Phase 5 above and adds several items not yet captured:

- **The project's own documentation contains an explicit self-correction.** `TUNING.md` §4
  literally says: *"Correction (2026-07-18 (6)/(7)): MPPI is not immune to this either. The line
  that used to be here claimed MPPI's trajectory-sampling approach doesn't share RPP's cusp
  failure mode — that turned out to be wrong, just live-verified too late to have caught it
  earlier."* This is direct, primary-source support for the user's point that this was never a
  clean, one-shot fix — even the project's own reference doc had to retract an earlier claim.
- **`GoalAngleCritic.cost_weight` raised 3.0→10.0 (2026-07-18 (5))** — fixes a *different* freeze:
  a goal that's mainly a reorientation (same/near position, new final yaw) wasn't moving at all.
  Root cause: `PathAlignCritic`/`PathFollowCritic`/`PathAngleCritic` all gate on distance from the
  robot's *current* pose to the goal, which is ~0 for this scenario from the first control cycle —
  disabling all three path-following critics for the entire maneuver, leaving only
  `GoalAngleCritic` active against `ConstraintCritic`'s turning-radius penalty. **Explicitly
  marked "not a complete fix"** even after the weight raise — still occasionally fails outright on
  retry-budget exhaustion, since MPPI's sampling is stochastic and the underlying critic-gating
  gap is unchanged. **This is the correct citation for the "final-goal reorientation freeze"
  already mentioned in Ch6's cusp-freeze section** (§sec:cusp-freeze) — I described this generically
  earlier in this session without a solid source; this is the real mechanism and it should be
  cited properly.
- **The mid-route cusp freeze (2026-07-18 (6)) is the same architectural issue as the later,
  fully-resolved cusp-freeze investigation already in Ch6.** `TUNING.md` names the exact same
  mechanism the thesis's existing §sec:cusp-freeze cites: `path_handler.cpp`'s
  `isWithinInversionTolerances()`, combined with `SmacPlannerHybrid` inserting a fresh small
  corrective reversal on nearly every replan. At the time `TUNING.md` was written (mid-July), this
  was **not yet fixed** — only properly root-caused and resolved with the dedicated
  `CuspStraightenerSmoother` over a month later (2026-08-25 onward). Strong confirmation these are
  the same underlying problem recurring at two points in the project's timeline, not two separate
  bugs — Ch6 should probably say so explicitly.
- **A statistical-rigor lesson (2026-07-19 (3))**: investigated raising `PathAlignCritic.cost_weight`
  to fix corner-cutting. Tried `14.0` (worse — overshot the other way) and `12.0` (same overshoot,
  not a graded response). A repeat run at the *default* `10.0` swung even wider on a different run
  of the identical goal — concluded MPPI's run-to-run stochasticity is large enough that a
  single-run weight experiment isn't reliable evidence, and deliberately left the parameter at
  default rather than commit to an untested value pending a proper multi-trial comparison. Good,
  honest example of recognizing insufficient evidence rather than shipping a plausible-looking fix.
- **`batch_size` raised 2000→2500/3500 (2026-07-19 (4))** — a genuine "real improvement, real cost"
  story: measurably fixed path-tracking looseness (p90 deviation `0.33-0.41m → 0.25-0.26m`,
  reproducible across trials) but degraded control-loop timing enough near the goal that one
  otherwise-normal run took **4.5 minutes** to converge instead of ~30s. Reverted for that cost,
  not because the fix didn't work.
- **The original cusp-freeze discovery, in full (2026-07-18 (6)), read directly from the
  changelog**: user reported the robot getting stuck on a path with a reversal-then-forward
  segment. Reproduced live on the same K-turn benchmark goal `(2.0, 1.0, 179°)` used throughout
  this arc — repeated `Failed to make progress` → abort → recovery → full replan, never reaching
  the goal across multiple 60-150s+ attempts. Ruled out as a regression from the same day's other
  fixes (reverted them live, reproduced identically). Root cause candidate identified — the exact
  same mechanism as Ch6's already-resolved investigation: `path_handler.cpp`'s
  `isWithinInversionTolerances()` requires satisfying both an xy and yaw tolerance at a cusp
  simultaneously before the rest of the path is released, and `SmacPlannerHybrid` was frequently
  inserting a *new* small corrective reversal at the start of nearly every fresh replan, each too
  short to satisfy `progress_checker`'s movement-time allowance, triggering another abort/replan
  cycle. Loosening the inversion tolerances live helped (more net progress per attempt) but did
  **not reliably fix it** within the test budget. **The user explicitly chose to pause the
  investigation here rather than keep iterating live** — it was deliberately left open, not
  abandoned by neglect. This is the true starting point of the cusp-freeze story: first found and
  clearly diagnosed 2026-07-18, left unresolved for over a month, then properly root-caused and
  fixed via the dedicated `CuspStraightenerSmoother` starting 2026-08-25. Ch6 should reflect this
  real timeline rather than presenting the August work as the start of the investigation.
- **A related, only-partially-resolved fix (2026-07-18 (7))**: fixed a narrower symptom — the
  robot not moving at all when the path's *first* segment needs a turn the start heading doesn't
  match. Root cause: `SmacPlannerHybrid`'s `change_penalty` defaulted to `0.0` (Nav2's real
  default, not a misconfiguration), so the planner had zero cost for inserting an unnecessary
  reversal as literally the first segment of a path. Fixed by raising it to `3.0`. **This surfaced
  a new trade-off, not fully resolved**: for large heading-change goals, the raised
  `change_penalty` combined with the already-raised `reverse_penalty` now sometimes pushes the
  planner toward a long forward-only loop instead of a short reverse — on the live test run, the
  robot followed the loop several meters past the goal, then stalled rotating in place for tens of
  seconds before the test was stopped. Explicitly flagged as the same underlying tension as the
  (6) freeze above: `reverse_penalty`/`change_penalty` discourage exactly the reversal/direction-change
  behavior this vehicle sometimes genuinely needs.
- **AMCL's `DifferentialMotionModel` confirmed correct for this Ackermann vehicle, not a bug** —
  checked directly against the Nav2 source; "Differential" here refers to how odometry noise is
  decomposed (rotate-translate-rotate), not the drivetrain type, and is the right choice for any
  non-holonomic robot, car-like or not. Nothing to fix, but worth a sentence confirming this was
  checked rather than assumed.

## Open correction (found 2026-08-30, not yet applied to thesis)
Ch6's already-written "curvature-continuing" cusp-extension refinement (added earlier this
session) describes a two-point curvature estimate that the changelog shows was buggy — it made
the extension curve too tightly and the robot deviated off-path. Fixed 2026-08-29 (same day) by
averaging curvature over a ~0.15m lookback window instead. The thesis currently describes the
buggy version. **Needs correction before this section is considered final.**

**CONFIRMED by `TUNING.md`/`PARAMS_REFERENCE.md`, both freshly rewritten 2026-08-31** (the user
updated these project files directly): the correction above is exactly right — both files now
state the windowed-curvature version explicitly and both say plainly, in so many words, **"not
yet hardware-tested"** for the curvature-continuing extension AND for `min_lead_distance`
(also added 2026-08-29). Ch6 must not present either refinement as validated beyond simulation.

## Phase 5c — `TUNING.md`/`PARAMS_REFERENCE.md` refreshed 2026-08-31 (user's own maintenance pass)
The user directly rewrote these two files today, catching drift between the docs and the live
config. Confirms everything in Phase 5/5b and adds:

- **§2a (`cusp_straightener`) is now written up authoritatively as "RESOLVED 2026-08-26"** for the
  mid-route cusp freeze, with the full trial numbers: baseline 110.2s/2 recoveries →
  `straight_distance` alone 86.7s/1.3 → `straight_distance`+`extend_distance` 43.9s/0 (final
  locked-in config, hardware-validated 2/2 goals, 0 recoveries). Explicitly states widening
  `inversion_xy_tolerance`/`inversion_yaw_tolerance` was tried again on 2026-08-25 and was
  **worse** (aborts before even reaching the cusp) — the smoother is what actually worked, not
  loosening MPPI's own gating.
- **A genuinely new, currently open, unresolved discrepancy in the live codebase**:
  `PathFollowCritic`/`PathAngleCritic.offset_from_furthest` was lowered from defaults (6/4) to
  (3/2) on 2026-07-18 to fix a "turning early" bug — but the live config is **currently back at
  the defaults (6/4)**, and no changelog entry documents when or why it was reverted. The user's
  own fresh audit flags this explicitly: *"treat this as a real, currently-unexplained discrepancy
  ... re-verify against the original 'turning early' symptom before trusting either value
  blindly."* **Do not present the "turning early" fix as a settled, currently-active fix in the
  thesis** — the value that supposedly fixed it is not actually deployed right now, for reasons
  nobody has recorded.
- The near-wall goal / `tolerance` silent-wrong-pose story (already accurately in Ch6) is now
  independently confirmed word-for-word in the project's own reference doc, including the same
  citation to the "prefer visible failure" project principle already used in Ch6.
- Minor: `max_planning_time` (5.0s) confirmed live (2026-08-29) to never actually be the limiting
  factor on a hard/infeasible goal — `max_iterations` is what's hit first (each failed attempt
  2.5-3.6s, well under the time cap).

---

## Coverage status (as of 2026-08-31)
All three CHANGELOG.md files (`qcar_initial`, `qcar_ackermann`, `qcar_updated`) now fully read —
confirmed `qcar_initial`/`qcar_ackermann` are identical copies of `qcar_updated`'s history up to
their respective rename points, no unique content. `TUNING.md` and a sample of
`PARAMS_REFERENCE.md` also read (the latter is a dry parameter reference with no unique
narrative beyond `TUNING.md`). All memory files read. `qcar_navigation` has no separate
CHANGELOG.md — its only history is the memory files already covered. Git log fully reviewed.
All 8 status PPTs and 3 chat summaries reviewed.

**This is believed to be complete coverage of the available written history.** Remaining
uncertainty is only in the user's own recollection of anything not captured in any of these
artifacts.
