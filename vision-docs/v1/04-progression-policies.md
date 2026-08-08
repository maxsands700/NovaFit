# Progression Policies

Progression happens at two separate levels:

1. Each exercise is evaluated independently after every exposure.
2. The whole program is evaluated from trends across exercises and movement categories.

Exercise A may progress while exercise B is maintained and exercise C is regressed.
One stalled exercise should not cause a whole-program deload.

The **Continuous Progression Model** below is the authoritative v1 decision
algorithm. The OG2-style discrete outcomes and examples above it define invariants,
fallback expectations, and golden test scenarios; they are not a second policy
implementation.

## Progression Loop

```
for scheduled_session in program:
  publish the current prescriptions
  athlete completes the prescribed session and logs the results

  after the session is complete:
    for exercise in scheduled_session:
      outcome = evaluate_exercise(completed_exposure)
      next_exercise_action = PROGRESS | MAINTAIN | REGRESS | STOP
      store the outcome, action, and reason

    if any outcome is PAIN_HOLD:
      place program in PAIN_HOLD
      do not publish another workout
      await correction or athlete confirmation
      return PAIN_HOLD

    program_action = review_program(program, completed_sessions)
    if program_action changes a prescription within its mutation envelope:
      create a prescription revision under the same program_id
    if program_action in [GOAL_ACHIEVED, GOAL_TARGET_DATE_REACHED, COMPLETE_PROGRAM]:
      return complete_program(program)
    if program_action == STOP_PROGRAM(reason=an early_end_reason):
      return ProgramEndReport to program generation

    publish the next scheduled session
```

## Exercise Exposure

```
evaluate_exercise(exposure):
  record published prescription, completed sets/reps, ending RPE/RIR, and pain
  require completed load == published prescribed load

  if athlete reports pain:
    stop the exercise
    # Normal exertion and ordinary DOMS are not pain.
    return PAIN_HOLD

  if prescribed work was completed:
    if ending_RIR > target_RIR:
      return EASY_PASS
    if ending_RIR == target_RIR:
      return PASS
    return HARD_PASS

  return FAIL
```

The default `target_RIR` is about 1. NovaFit assumes the athlete uses the published
load, reports RIR honestly, and performs all submitted reps with good form. Form is
not an athlete input or an evaluated signal in v1.

## Pain Hold and Corrections

A pain report on a completed set or a `SKIPPED(PAIN)` row immediately puts the
program in `PAIN_HOLD`. The athlete stops the affected exercise; NovaFit makes no
substitution, rehabilitation prescription, or diagnosis, and publishes no next
workout while the hold remains.

The athlete may submit an immutable correction if the pain entry was accidental.
NovaFit recomputes every affected exercise and program decision from the corrected
revision, retaining the original log and superseded decisions for audit. If no pain
report remains and all required evidence is valid, NovaFit releases the hold and
continues the existing program. If the athlete confirms that the pain report stands,
NovaFit ends the program with `PAIN_REPORTED`; the athlete must be ready to train
safely again before beginning a new evidence-based program.

## Exercise-level Decision

```
decide_next_exposure(exercise, recent_exposures, athlete):
  state = update_continuous_exercise_evidence(exercise, recent_exposures)
  return continuous_exercise_decision(state, exercise.current_prescription)
```

The authoritative decision retains the OG2 behavior that one unsuccessful exposure
normally causes `MAINTAIN`, while repeated supported regression evidence causes
`REGRESS_ONE_VARIABLE`.

Regression should make the smallest useful change:

```
regress_one_variable(exercise):
  if load was recently increased:
    remove the last load increase
  else if reps can be reduced within the programmed range:
    reduce reps
  else if sets can be reduced within the program's mutation envelope:
    reduce one set
    if exercise.role == PRIMARY and exercise.set_count == 2:
      exercise.role = SECONDARY
      # Store the evidence and reason for the role change. Preserve all other variables.
  else if a previous exercise exists in the declared progression_path:
    regress to that exercise  # future calisthenics support
  else:
    return STOP_PROGRAM(EXERCISE_REGRESSION_BOUNDARY_REACHED)

  preserve every other variable
```

## Progression Cadence Priors

Beginner and intermediate are initial priors, not permanent switches. A beginner
exercise starts with an adaptation-rate prior of `1.0`, favoring workout-to-workout
progression. An intermediate exercise starts at `0.5`, normally requiring about two
successful exposures. The continuous model updates that rate independently for each
exercise from observed responses to overload.

For pull-ups and dips, `load` means external load. Valid increments come from the
athlete's available equipment. If more detailed inventory is unavailable, the v1
defaults are `5 lb` total for barbell and `5 lb` per hand (`10 lb` aggregate) for a
paired-dumbbell exercise, as defined in `08-exercise-catalog.md`.

### Advanced

Advanced progression should generally be planned across a mesocycle rather than
applied automatically after every workout.

```
progress_advanced(exercise):
  follow planned volume / intensity progression for the current mesocycle
  use each exposure to decide MAINTAIN or REGRESS
  add load or volume only when prescribed by the mesocycle
  continue through internal phases declared by the program
  when the declared program is complete, return COMPLETE_PROGRAM
```

Detailed advanced periodization is outside the first progression-policy version.

## Program-level Review

The program review uses the authoritative continuous aggregation below. It preserves
isolated exercise decisions and escalates only coordinated, sufficiently confident
decline or stall across primary exercises and movement categories.

NovaFit does not measure or diagnose CNS fatigue. A program-level deload is inferred
only from coordinated performance and RPE trends. Sleep, readiness, and recovery
data remain outside v1.

## Program-level Actions

```
performance_triggered_deload(program):
  require program.performance_triggered_deload_count < 1
  use 1–2 light sessions over 7 days
  retain the highest-priority declared exercise in each represented movement category
  prescribe 1–2 sets at anchor reps and a load supporting at least 4 RIR
  preserve technique and movement familiarity
  increment program.performance_triggered_deload_count

  after deload:
    if program.retest_policy.trigger == AFTER_EVERY_DELOAD:
      return capability_retest_and_stop(program)
    resume from the last stable prescription
    create a prescription revision under the same program_id

capability_retest_and_stop(program):
  for progression_track in program.capability_retests:
    athlete performs the declared capability_retest_protocol
    record exercise, load, reps, RPE/RIR, and pain
    update the athlete's CurrentCapability with the dated test result

  recompute exercise and program evidence from the new capabilities
  stop the current program
  return ProgramEndReport(CAPABILITY_RETEST_COMPLETE) to program generation

complete_program(program):
  complete the final deload / taper
  return capability_retest_and_stop(program)
```

Every program declares a retest policy:

```
RetestPolicy {
  trigger: AFTER_EVERY_DELOAD | AT_PROGRAM_END
  after_retest: STOP_AND_REGENERATE
}
```

Simple beginner and intermediate programs default to `AFTER_EVERY_DELOAD`. Advanced
periodized programs default to `AT_PROGRAM_END`, allowing planned internal deloads
without forcing a maximal test. Any completed retest ends the current program and
returns control to program generation. Each program permits one performance-triggered
deload; a planned advanced deload does not count toward that limit.

The athlete remains responsible for warming up, progresses from easy attempts, rests
3–5 minutes before the test, and stops before technique breaks down. The test uses
the success measure declared by the program; it is stored as a capability assessment,
not as an ordinary training exposure.

## Capability Testing and Estimated 1RM

A formal capability assessment should use one clean near-maximal performance, not
accumulated work across multiple sets. Prefer a true 1RM when appropriate. Otherwise,
use a near-maximal low-rep set or test the goal's declared success measure.

A true 1RM is recorded directly. Estimate 1RM from individual working sets as follows:

```
Brzycki(load, reps) = load * 36 / (37 - reps)
Epley(load, reps)   = load * (1 + reps / 30)

estimate_1RM(load, reps):
  if reps < 8:
    return Brzycki(load, reps)

  if 8 <= reps <= 10:
    interpolation = (reps - 8) / 2
    return Brzycki(load, reps) * (1 - interpolation)
         + Epley(load, reps) * interpolation

  return Epley(load, reps)
```

This uses 100% Brzycki at 8 reps, a 50/50 blend at 9 reps, and 100% Epley at
10 reps. For a submaximal working set:

```
raw_e1RM = estimate_1RM(load, completed_reps)
RIR_adjusted_e1RM = estimate_1RM(load, completed_reps + reported_RIR)
```

Use the best valid clean `RIR_adjusted_e1RM` as the session's estimated-strength
observation. Retain the raw value for auditing. RPE and RIR are mandatory for every
completed work-set observation.

For pull-ups and dips, estimate from total system load (`bodyweight + external_load`),
then subtract bodyweight when reporting the equivalent external load. Dumbbell loads
must use the canonical catalog's consistent per-hand or total-load convention.

Initial load calculations and the initial mutation envelope belong to program
generation. Standard load increments and progression parameters belong to this
policy.

---
# Continuous Progression Model — Authoritative v1 Policy

V1 uses a deterministic continuous evidence model with discrete actions:
`PROGRESS`, `MAINTAIN`, `REGRESS`, `DELOAD`, or `STOP`. OG2 heuristics supply initial
priors, hard boundaries, and fallback scenarios. This is not a learned model; every
input, contribution, threshold, state update, and decision is versioned and stored.

Pain remains a hard stop outside the score. Sleep, readiness, and recovery data are
not v1 inputs.

## Comparable Exposures

Two exposures are **prescription-comparable** when they use the same exercise,
role, set count, rep target, load, rest, and tempo. Use them for consecutive-failure
and same-work effort comparisons.

Two exposures are **strength-comparable** when they use the same exercise, role,
tempo, rep range, and progression track, and differ only through permitted rep or
load mutations. Use RIR-adjusted e1RM to compare them. A set-count or role change
starts a new comparison window. Capability tests, deload work, undeclared work,
invalid logs, and non-performance skips are not training exposures.

Use at most the previous 12 comparable exposures from the previous 42 days. Within
that window, multiply each older observation's weight by `0.8` for every newer
comparable exposure. A valid complete log has weight `1.0`; a non-comparable
observation has weight `0`.

## Exercise Evidence State and Signals

```
ExerciseEvidenceState {
  progress_credit             # [0, 1.5]
  regression_pressure         # [0, 1]
  confidence                  # [0, 1]
  adaptation_rate             # [0.25, 1]
  consecutive_underperformances
  comparable_exposure_history
  last_action
}
```

For each comparable exposure:

```
completion_ratio = completed_prescribed_reps / prescribed_reps
effort_delta = reported_RIR - target_RIR
strength_delta = change in RIR_adjusted_e1RM versus the weighted median
                 of the previous 3 strength-comparable exposures

positive_strength = clamp((strength_delta - 0.01) / 0.02, 0, 1)
success_evidence = completion_ratio
                   * clamp(1 + 0.5 * effort_delta
                             + 0.25 * positive_strength, 0, 1.25)

incompletion = 0                                      if completion_ratio == 1
               0.5 + 0.5 * (1 - completion_ratio)   otherwise
overexertion = clamp((target_RIR - reported_RIR) / 2, 0, 1)
strength_decline = clamp((-strength_delta - 0.01) / 0.02, 0, 1)

regression_observation = 0.60 * incompletion
                         + 0.25 * overexertion
                         + 0.15 * strength_decline
```

Treat e1RM movement inside ±1% as noise. RIR is the effort input; RPE is mandatory
corroborating data and is never counted as a second effort signal.

For every completed work set, require `RPE` and `RIR` and validate:

```
expected_RPE = 10 - reported_RIR
RPE_RIR_consistent = abs(reported_RPE - expected_RPE) <= 0.5
```

If either value is absent or the pair is inconsistent, store the submission as
`NEEDS_CORRECTION(MISSING_EFFORT_DATA | RPE_RIR_MISMATCH)`. Do not update evidence or
publish the next prescription until the athlete submits an auditable correction.

```
if exposure is complete:
  progress_credit = min(1.5,
    progress_credit + adaptation_rate * observation_weight * success_evidence)
else:
  progress_credit = 0.5 * progress_credit

regression_pressure = min(1,
  0.6 * regression_pressure + observation_weight * regression_observation)

effective_observations = 1 current direct capability assessment
                         + sum(recency_adjusted_observation_weights)
confidence = 1 - exp(-effective_observations / 2)
```

An underperformance is incomplete work or completion at least one RIR harder than
prescribed. Missing data does not count as underperformance.

## Exercise Decision and Hysteresis

```
continuous_exercise_decision(state, latest_exposure):
  if pain was reported:
    return STOP

  if confidence < 0.60:
    return MAINTAIN

  if regression_pressure >= 0.65
     and consecutive_underperformances >= 2:
    return REGRESS_ONE_VARIABLE

  if progress_credit >= 1.0
     and regression_pressure <= 0.25
     and latest exposure was complete at target RIR or easier:
    return PROGRESS_ONE_VARIABLE

  return MAINTAIN
```

After progression, subtract `1.0` from progress credit. After regression, set
progress credit to `0`, regression pressure to `0.25`, and require a new comparable
exposure before another mutation. Regression wins if both action thresholds are
somehow satisfied. This is the v1 hysteresis rule.

## Continuous Cadence

Beginner and intermediate labels seed, but do not control, progression cadence. A
beginner exercise starts with `adaptation_rate = 1.0`; an intermediate exercise
starts at `0.5`. After each prescribed overload, update the rate from the first
comparable exposure at the new prescription:

```
overload_outcome = 1.0   if complete at target RIR or easier
                   0.25  if complete materially harder than target
                   0.0   if incomplete

adaptation_rate = clamp(0.8 * adaptation_rate
                        + 0.2 * overload_outcome, 0.25, 1.0)
```

Repeated successful overload moves an exercise toward workout-to-workout progress;
difficulty absorbing overload slows it toward one change per several exposures.
Advanced progression follows its declared mesocycle; continuous evidence may
maintain or regress it but may not add unplanned overload.

## Load-Aware Double Progression

Every exercise declares an `anchor_rep_count`, normally its initial baseline and
usually 5 for a primary compound exercise. Once the continuous decision is
`PROGRESS`, NovaFit chooses the largest conservative supported rep step, while
capping the automatic change at one additional rep per set per exposure.

```
rep_surplus = floor(min_recorded_RIR - target_RIR)

if rep_surplus >= 1:
  rep_candidate = add 1 rep to every set, bounded by the rep-range ceiling
else:
  rep_candidate = add 1 total rep in balanced order
  # 5/5/5 -> 6/5/5 -> 6/6/5 -> 6/6/6
```

Before selecting the rep candidate, test whether the smallest available load
increment is already supported at the anchor:

```
next_load = current_load + smallest_available_increment
required_e1RM = estimate_1RM(next_load, anchor_rep_count + target_RIR)
conservative_session_e1RM = lowest valid RIR_adjusted_e1RM among work sets,
                            or the final/hardest set when only ending RIR exists

load_step_supported = conservative_session_e1RM >= required_e1RM * 1.01

choose_progression_step(exercise, exposure, state):
  require continuous_exercise_decision(...) == PROGRESS_ONE_VARIABLE

  if load_step_supported:
    return ADVANCE_LOAD_TIER(next_load, reset_all_sets_to=anchor_rep_count)
  if rep_surplus >= 1:
    return INCREASE_EACH_SET_BY_ONE_REP
  return INCREASE_ONE_TOTAL_REP
```

`ADVANCE_LOAD_TIER` is one atomic load mutation; its rep reset is not a second
overload variable. The rep-range ceiling is a boundary, not a required destination.
This returns work to the baseline rep schema as soon as the next load tier is
supported and avoids accumulating unnecessary rep volume.

For catalog exercises that support only `BODYWEIGHT_REPS`, skip the load-tier test
and progress reps within the declared envelope. If the athlete reaches its upper
boundary without completing the goal, return
`STOP_PROGRAM(EXERCISE_PROGRESSION_BOUNDARY_REACHED)` so the athlete can retest and
program generation can create a new evidence-based program. V1 does not silently
add external load to a rep-only exercise.

## Continuous Program Evidence

`Primary` means an exercise whose current program role is `PRIMARY`. An affected
exercise has `regression_pressure >= 0.50`. A meaningful effort rise is at least
`+1 RPE` or `-1 RIR` for prescription-comparable work.

```
breadth = affected_primary_exercises / primary_exercises
categories = min(affected_movement_categories / 2, 1)
synchrony = proportion of affected exercises declining in the same
            two-comparable-exposure window
magnitude = mean regression_pressure among affected exercises

program_regression = 0.35 * breadth
                     + 0.25 * categories
                     + 0.20 * synchrony
                     + 0.20 * magnitude
```

Trigger `PERFORMANCE_TRIGGERED_DELOAD` when `program_regression >= 0.70`, breadth is
at least `0.50`, at least two movement categories are affected, the decline persists
across two comparable exposures, meaningful effort rises for the same or less work,
and mean affected-exercise confidence is at least `0.60`. Otherwise preserve the
program and apply only exercise-level actions.

A measurable-progress event is a successfully retained rep or load increase, an
e1RM improvement greater than 1%, or an improvement of at least one RIR at the same
work. `Most exercises` means more than half of current primary exercises. Determine
the broad-stall horizon continuously:

```
rate = median adaptation_rate across primary exercises
stall_horizon_days = 28 - 21 * clamp((rate - 0.25) / 0.50, 0, 1)
```

Trigger a performance deload when most primary exercises across at least two
movement categories have no measurable-progress event for that horizon and program
confidence is at least `0.60`. If the same broad regression or stall returns after
the program's one permitted performance-triggered deload, stop and regenerate.
Planned advanced deloads do not count toward that limit.

Every program-level score must retain its breadth, categories, synchrony, magnitude,
persistence, confidence, and source observations. A combined score must never hide
why the program changed.

## OG2 Policy Boundary

The continuous model decides when to act; OG2 constrains the permitted action. It
must honor target RIR, declared volume and mutation bounds, the smallest available
load increment, the rule to change one overload variable at a time, and the ban on
unplanned advanced overload. Policy constants may change only in a new immutable
policy version with updated golden scenarios.

---
# Progression Layer Contract

## 1. Program Completion and Mutation Declaration

Program generation must declare completion conditions and mutation limits when it
creates a program. A normal completion is either a verified active-goal achievement,
an active goal's target date, or the declared maximum program duration. At a normal
completion, NovaFit completes the declared deload/taper when applicable, performs
the declared capability retests, records the outcome of every active goal, and
returns control to program generation.

An early end is not a medical diagnosis. It records the observed program outcome and
its evidence: a confirmed pain report, persistent broad regression after the
permitted deload, broad stall, a mutation boundary, changed constraints, or an
athlete-requested end.
Exercise-level changes and a performance-triggered deload must be attempted only as
allowed by the mutation envelope before an evidence-led early end.

```
ProgramCompletionPolicy {
  maximum_duration_weeks
  active_goal_targets       # exercise, success measure, target value, target date
  completion_retest_policy
  deload_policy

  normal_end_reasons: [GOAL_ACHIEVED, GOAL_TARGET_DATE_REACHED, COMPLETE_PROGRAM]
  early_end_reasons: [PAIN_REPORTED, BROAD_REGRESSION, BROAD_STALL,
                      PROGRESSION_BOUNDARY_REACHED, PROGRESSION_PATH_EXHAUSTED,
                      PROGRAM_CONSTRAINTS_CHANGED, ATHLETE_REQUESTED_END]
}
```

## 2. Program Mutation Envelope

Program generation must declare the changes that progression policy may make while
preserving the program's identity.

```
ProgramMutationEnvelope {
  program_id
  policy_version
  completion_policy

  fixed: [goals, exact_exercises, weekly_structure, movement_allocation, schedule]
  permitted_variables: [reps, load, sets, role]
  permitted_role_transition: PRIMARY_TO_SECONDARY_ONLY

  for each progression_track:
    modality_policy: FIXED_EXERCISE | DECLARED_PROGRESSION_PATH
    current_exercise
    rep_range
    set_range
    load_range
    permitted_load_increments
    progression_method
    permitted_exercise_transitions
    capability_retest_protocol
}
```

Within a v1 track, reps may change only within its declared rep range, load only by
its declared equipment increments and within its declared load bounds, and set count
only between three primary sets and two secondary sets. Each revision changes one
variable at a time and stores its evidence and reason. Changing a fixed field,
selecting an undeclared exercise, or exceeding a declared bound ends the program;
the factual `ProgramEndReport` is the only input to regeneration.

V1 general strength uses `FIXED_EXERCISE`. Bench press remains bench press;
NovaFit may change its reps, load, or sets only within the declared bounds. Pull-ups
and dips also retain their exercise identity while external load changes.

Future calisthenics programs may use `DECLARED_PROGRESSION_PATH`:

```
ProgressionPath {
  progression_track: vertical_push
  nodes: [chest_to_wall_HSPU, freestanding_HSPU_negative, ...]
  directed_edges: permitted progressions and regressions
  transition_requirements: evidence required for each edge
}
```

Moving along a path declared by the original program is progression within that
program. Selecting an exercise or transition not declared in the path ends the
program and returns control to program generation. This allows future modalities
without permitting arbitrary exercise substitution.

Every allowed change creates a new prescription revision under the same
`program_id`. It does not create a new program.

## 3. Program-ending Conditions

```
should_stop_program(program, evidence):
  if program.status == PAIN_HOLD and athlete confirms pain_report_stands:
    return PAIN_REPORTED

  if a declared capability retest verifies any active goal target:
    return GOAL_ACHIEVED

  if an active goal target date has arrived:
    return GOAL_TARGET_DATE_REACHED

  if progression or regression reaches the mutation envelope boundary:
    return PROGRESSION_BOUNDARY_REACHED

  if a future progression_path has no permitted next transition:
    return PROGRESSION_PATH_EXHAUSTED

  if broad regression persists after the permitted deload:
    return BROAD_REGRESSION

  if broad stall reaches sufficient evidence and confidence:
    return BROAD_STALL

  if program.completion_policy.maximum_duration_weeks is complete:
    return COMPLETE_PROGRAM

  if goals, equipment, availability, or other program constraints change:
    return PROGRAM_CONSTRAINTS_CHANGED

  if the athlete explicitly ends the program:
    return ATHLETE_REQUESTED_END

  return CONTINUE_PROGRAM
```

A stop reason describes what happened; it does not prescribe the replacement.

## 4. Handoff Contract

When a program stops, progression policy returns a factual report to program
generation:

```
ProgramEndReport {
  program_id
  final_prescription_revision
  started_at
  ended_at
  reason
  policy_version

  supporting_evidence
  actions_already_applied
  capability_retest_results
  final_exercise_evidence
  final_progression_path_positions
  goal_results
  current_capabilities
  changed_constraints
}
```

The report must contain the evidence and policy decisions that caused the stop. It
must not choose replacement exercises, restructure the schedule, or generate the
next program.

## 5. Continuous Model Parameters

The continuous model must be driven by a versioned policy configuration rather than
hard-coded constants:

```
ProgressionPolicyConfig {
  policy_version

  recency_decay_per_exposure: 0.8
  comparable_exposure_limit: 12
  comparable_max_age_days: 42
  e1RM_noise_band: 0.01
  confidence_threshold: 0.60

  success_effort_weight: 0.50
  success_strength_weight: 0.25
  regression_completion_weight: 0.60
  regression_effort_weight: 0.25
  regression_strength_weight: 0.15

  progress_credit_threshold: 1.0
  progress_max_regression_pressure: 0.25
  regression_pressure_threshold: 0.65
  required_consecutive_underperformances: 2
  post_regression_pressure: 0.25

  beginner_adaptation_rate_prior: 1.0
  intermediate_adaptation_rate_prior: 0.5
  adaptation_rate_floor: 0.25
  adaptation_rate_update_old_weight: 0.8
  adaptation_rate_update_new_weight: 0.2

  max_rep_increase_per_set_per_exposure: 1
  load_step_e1RM_support_margin: 1.01

  affected_exercise_pressure: 0.50
  program_breadth_weight: 0.35
  program_category_weight: 0.25
  program_synchrony_weight: 0.20
  program_magnitude_weight: 0.20
  program_regression_threshold: 0.70
  minimum_affected_categories: 2
  meaningful_RPE_change: 1
  meaningful_RIR_change: 1
  performance_triggered_deload_limit: 1

  fallback_OG2_heuristics
}
```

These are the initial immutable v1 values. Athlete-specific data changes evidence
state and progression cadence, but never silently changes this configuration. All
inputs, contributions, thresholds, and outputs are stored with each decision.

## 6. Logging Edge Cases

```
classify_logged_exposure(log):
  if a required capability retest is unlogged or invalid:
    return WAITING_FOR_VALID_RETEST  # do not resume the program

  if the workout was never submitted:
    return NO_OBSERVATION  # do not treat missing data as failure

  require every submitted set row has outcome COMPLETED or SKIPPED

  if any row is SKIPPED with reason PAIN or reports pain:
    return PAIN_HOLD

  if any completed row has completed_reps > prescribed_reps:
    return NEEDS_CORRECTION(UNDECLARED_REPS)

  if any completed row has completed_load != published_prescribed_load:
    return NEEDS_CORRECTION(UNDECLARED_LOAD)

  if any COMPLETED row is missing completed_reps, RPE, or RIR:
    return NEEDS_CORRECTION(MISSING_EFFORT_DATA)

  if any COMPLETED row has abs(RPE - (10 - RIR)) > 0.5:
    return NEEDS_CORRECTION(RPE_RIR_MISMATCH)

  if any COMPLETED row has a skip reason:
    return NEEDS_CORRECTION(INVALID_LOG)

  if any SKIPPED row has no reason, populated completion fields, or checked Pain:
    return NEEDS_CORRECTION(INVALID_LOG)

  if any COMPLETED row has completed_reps < prescribed_reps:
    return FAIL

  if any row is SKIPPED with reason PERFORMANCE_LIMIT:
    return FAIL

  if any row is SKIPPED with reason NON_PERFORMANCE:
    return NON_COMPARABLE  # maintain and lower confidence

  if the athlete performed an undeclared exercise:
    return NON_COMPARABLE  # do not infer progress or regression

  # Good form is assumed under the athlete disclaimer.
  return COMPARABLE_EXPOSURE
```

Partial workouts are evaluated exercise by exercise. An exercise updates evidence
only when all of its prescribed rows are `COMPLETED`; another exercise may be
skipped, failed, or held without invalidating that completed exercise's evidence. A
submitted partial workout is valid only when every remaining set is explicitly
`SKIPPED` with a reason. Late log corrections recompute affected evidence and create
an auditable replacement decision; previous decisions are preserved. A log awaiting
mandatory correction or a `PAIN_HOLD` blocks the next prescription for that program.
