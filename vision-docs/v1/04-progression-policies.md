# Progression Policies

Progression happens at two separate levels:

1. Each exercise is evaluated independently after every exposure.
2. The whole program is evaluated from trends across exercises and movement categories.

Exercise A may progress while exercise B is maintained and exercise C is regressed.
One stalled exercise should not cause a whole-program deload.

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
  record prescribed and completed sets, reps, load, ending RPE/RIR, and technique

  if athlete reports pain:
    stop the exercise
    # Normal exertion and ordinary DOMS are not pain.
    return STOP_AND_SEEK_SAFE_GUIDANCE

  if prescribed work was completed with clean technique:
    if ending_RIR > target_RIR:
      return EASY_PASS
    if ending_RIR == target_RIR:
      return PASS
    return HARD_PASS

  return FAIL
```

The default `target_RIR` is about 1. NovaFit assumes the athlete reports RIR and
technique honestly.

## Exercise-level Decision

```
decide_next_exposure(exercise, recent_exposures, athlete):
  latest = evaluate_exercise(recent_exposures.latest)
  stage = progression_stage(athlete, exercise)

  if latest == STOP_AND_SEEK_SAFE_GUIDANCE:
    return STOP

  if latest in [EASY_PASS, PASS]:
    return progression_for_training_level(exercise, stage)

  if latest == HARD_PASS:
    return MAINTAIN

  if latest == FAIL and previous exposure was not FAIL:
    return MAINTAIN  # repeat once before changing the prescription

  if latest == FAIL and previous exposure was FAIL:
    return REGRESS_ONE_VARIABLE
```

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

## Progression by Training Level

Use the simplest progression that still produces progress.

Progression level is exercise-specific. An athlete may be a beginner on one lift
and intermediate or advanced on another. Use the athlete's onboarding level as the
initial default, then update each exercise from its observed rate of progress:

```
progression_stage(athlete, exercise):
  BEGINNER = can progress the exercise workout to workout
  INTERMEDIATE = can progress the exercise every few workouts or weekly
  ADVANCED = requires planned mesocycle progression
```

### Beginner

Beginners should generally progress workout to workout using double progression.

```
progress_beginner(exercise):
  if not all sets have reached top_of_rep_range:
    add 1 total rep to the exercise
    # Example: 5/5/5 -> 6/5/5 -> 6/6/5 -> 6/6/6.
  else:
    advance_progression_track(exercise)
```

For weighted-bodyweight exercises, `load` means external load. Dumbbell and barbell
increments depend on the equipment available and the exercise.

```
advance_progression_track(exercise):
  if modality_policy == FIXED_EXERCISE
     and the next load increment is permitted by the mutation envelope:
    add smallest_available_load_increment(exercise)
    reset all sets to bottom_of_rep_range

  else if modality_policy == DECLARED_PROGRESSION_PATH
          and progression evidence satisfies a declared outgoing edge:
    move to that edge's next exercise
    reset the prescription as declared by the edge

  else:
    return STOP_PROGRAM(PROGRESSION_BOUNDARY_REACHED)
```

### Intermediate

Intermediates should generally progress every few workouts or weekly.

```
progress_intermediate(exercise):
  require 2 successful exposures at the current prescription

  if not all sets have reached top_of_rep_range:
    add 1 total rep to the exercise
  else:
    advance_progression_track(exercise)
```

If this no longer works within the program's mutation envelope, flag the program to
stop rather than adding multiple progression variables at once.

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

The program review aggregates exercise decisions without replacing them.

```
review_program(program, completed_sessions):
  for exercise in program.exercises:
    exercise_decision[exercise] = decide_next_exposure(exercise)

  if any exercise_decision == STOP:
    return STOP_PROGRAM(SAFETY_STOP)

  if decline is isolated to one exercise or one movement category:
    apply only its exercise-level MAINTAIN or REGRESS decision
    return CONTINUE_PROGRAM

  broad_regression = (
    at least half of primary exercises regress
    and regression spans at least 2 movement categories
    and affected exercises regress across 2 consecutive comparable exposures
  )

  if broad_regression and RPE rises for the same or less work:
    return PERFORMANCE_TRIGGERED_DELOAD

  broad_stall = most primary exercises make no progress
  if primary exercises are mostly BEGINNER and broad_stall lasts about 1 week:
    return PERFORMANCE_TRIGGERED_DELOAD
  if primary exercises are mostly INTERMEDIATE or ADVANCED
     and broad_stall lasts about 4 weeks:
    return PERFORMANCE_TRIGGERED_DELOAD

  return CONTINUE_PROGRAM
```

NovaFit does not measure or diagnose CNS fatigue. A program-level deload is inferred
only from coordinated performance and RPE trends. Sleep, readiness, and recovery
data remain outside v1.

## Program-level Actions

```
performance_triggered_deload(program):
  use 1–2 light sessions for the week
  retain one push, one pull, and one leg exercise
  prescribe 1–2 easy sets per exercise, well away from failure
  preserve technique and movement familiarity

  after deload:
    if program.retest_policy.trigger == AFTER_EVERY_DELOAD:
      return capability_retest_and_stop(program)
    resume from the last stable prescription
    create a prescription revision under the same program_id

capability_retest_and_stop(program):
  for progression_track in program.capability_retests:
    athlete performs the declared capability_retest_protocol
    record exercise, load, reps, RPE/RIR, technique, and pain
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
returns control to program generation.

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
observation. Retain the raw value for auditing. If RIR is missing, use the raw value
with lower confidence.

For weighted pull-ups, estimate from total system load (`bodyweight + external_load`),
then subtract bodyweight when reporting the equivalent external load. Dumbbell loads
must use the exercise library's consistent per-hand or total-load convention.

Initial load calculations and the initial mutation envelope belong to program
generation. Standard load increments and progression parameters belong to this
policy.

---
# Continuous Progression Model

The rules above describe the OG2 heuristics that NovaFit must satisfy. They should
not be implemented as rigid beginner/intermediate/advanced switches. NovaFit should
accumulate evidence continuously, then use that evidence to choose the discrete
next action: `PROGRESS`, `MAINTAIN`, `REGRESS`, or `STOP`.

The heuristic timeframes above are initial priors and fallbacks, not hard boundaries.
As the athlete logs more comparable exposures, their own data should increasingly
determine the progression cadence.

## Exercise Evidence State

Each exercise maintains independent bounded `[0, 1]` evidence values:

```
ExerciseEvidence {
  progression_evidence  # evidence that more overload is appropriate
  regression_evidence   # evidence that the current prescription is excessive
  confidence            # quantity and consistency of relevant observations
  progression_rate      # observed rate of improvement across exposures
}
```

These values are decision signals, not probabilities. Every update must retain the
observations and weights that produced it so NovaFit can explain the decision.

```
update_exercise_evidence(state, completed_exposure):
  observations = normalize(
    completion relative to prescription,
    ending RIR relative to target RIR,
    reps and load relative to comparable exposures,
    estimated strength trend,
    technique status
  )

  weight recent comparable exposures more heavily
  update progression_evidence
  update regression_evidence
  update confidence from evidence quantity and consistency
  update progression_rate from successful overload over time

  return state with an explanation of each contribution
```

Pain remains a hard stop and is not softened into a score. Sleep, readiness, and
recovery data are not inputs in v1.

## Continuous Exercise Decision

```
decide_next_exposure(state, current_prescription):
  if pain was reported:
    return STOP

  if state.confidence is insufficient:
    return MAINTAIN

  if state.progression_evidence is high
     and state.regression_evidence is low:
    return PROGRESS_ONE_VARIABLE

  if state.regression_evidence is high:
    return REGRESS_ONE_VARIABLE

  return MAINTAIN
```

Use hysteresis: require stronger evidence to change state than to remain in the
current state. This prevents repeated `PROGRESS` / `REGRESS` oscillation around a
single threshold.

## Continuous Progression Cadence

Beginner, intermediate, and advanced remain useful descriptions, but the underlying
policy should use the exercise's observed `progression_rate`:

```
select_progression_policy(exercise_evidence):
  faster progression_rate:
    favor workout-to-workout double progression

  moderate progression_rate:
    accumulate evidence across several exposures before progressing

  slower progression_rate:
    favor planned progression across the mesocycle
```

An athlete may therefore move gradually between progression methods, and may use a
different method for each exercise. The method must still follow OG2: use the
simplest progression that works and change one variable at a time.

## Continuous Program Evidence

Program-level analysis aggregates exercise evidence without averaging away its
structure. It should consider:

* Proportion of primary exercises declining
* Number of affected movement categories
* Magnitude of performance and RPE change
* Synchrony of decline across comparable exposures
* Persistence of the trend
* Confidence in the underlying exercise evidence

```
update_program_evidence(program, exercise_states):
  local_decline = regression is isolated to an exercise or movement category
  breadth = proportion of primary exercises and movement categories affected
  synchrony = degree to which declines occur together
  persistence = recency-weighted duration of the pattern

  program_regression_evidence = combine(breadth, synchrony, persistence, magnitude)
  program_confidence = combine(exercise confidence, comparable exposure count)

  if local_decline:
    preserve the program and apply only exercise-level decisions
  else if program_regression_evidence is high and program_confidence is sufficient:
    apply PERFORMANCE_TRIGGERED_DELOAD
  else:
    continue accumulating evidence
```

A single score must never hide why the program changed. NovaFit should store and
communicate whether the decision came from breadth, magnitude, synchrony,
persistence, or a combination of them.

## OG2 Policy Boundary

The continuous model decides when to act; OG2 heuristics constrain what actions are
allowed. It must honor declared goals and any approved structural-balance override,
target RIR, volume ranges, minimum useful exercise volume, and the rule to change
one progression variable at a time. All score definitions, weights, thresholds, and
changes to them must be deterministic, versioned, and auditable.

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
its evidence: a safety stop, persistent broad regression after the permitted deload,
broad stall, a mutation boundary, changed constraints, or an athlete-requested end.
Exercise-level changes and a performance-triggered deload must be attempted only as
allowed by the mutation envelope before an evidence-led early end.

```
ProgramCompletionPolicy {
  maximum_duration_weeks
  active_goal_targets       # exercise, success measure, target value, target date
  completion_retest_policy
  deload_policy

  normal_end_reasons: [GOAL_ACHIEVED, GOAL_TARGET_DATE_REACHED, COMPLETE_PROGRAM]
  early_end_reasons: [SAFETY_STOP, BROAD_REGRESSION, BROAD_STALL,
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

V1 general weighted strength uses `FIXED_EXERCISE`. Bench press remains bench press;
NovaFit may change its reps, load, or sets only within the declared bounds. Weighted
pull-ups also retain their exercise identity while external load changes.

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
  if athlete reports pain requiring the programmed work to stop:
    return SAFETY_STOP

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
  signal_weights: {
    completion
    RIR_delta
    reps_and_load_trend
    estimated_strength_trend
    explicit_technique_breakdown
  }

  recency_decay
  comparable_exposure_window
  minimum_confidence
  progression_threshold
  regression_threshold
  hysteresis_margin

  program_breadth_weight
  program_synchrony_weight
  program_persistence_weight
  program_magnitude_weight
  deload_limit

  fallback_OG2_heuristics
}
```

The initial values should implement the OG2 heuristics above. Athlete-specific data
may change evidence values and progression cadence, but must not silently change the
policy configuration. All inputs, contributions, thresholds, and outputs must be
stored with the decision.

## 6. Logging Edge Cases

```
classify_logged_exposure(log):
  if a required capability retest is unlogged or invalid:
    return WAITING_FOR_VALID_RETEST  # do not resume the program

  if the entire workout or exercise is unlogged:
    return NO_OBSERVATION  # do not treat missing data as failure

  if the log is invalid or internally inconsistent:
    return NEEDS_CORRECTION

  if the athlete skipped the exercise because of pain:
    return STOP_AND_SEEK_SAFE_GUIDANCE

  if the athlete attempted but could not complete the prescription:
    return FAIL

  if the exercise was skipped for a non-performance reason:
    return NON_COMPARABLE  # maintain and lower confidence

  if the athlete performed an undeclared exercise or prescription:
    return NON_COMPARABLE  # do not infer progress or regression

  if sets, reps, and load are complete but RIR is missing:
    use completion evidence only and lower confidence

  if the athlete explicitly reports technique breakdown:
    return FAIL

  # Otherwise clean technique is assumed under the athlete disclaimer.
  return COMPARABLE_EXPOSURE
```

Partial workouts must be evaluated exercise by exercise. A valid completed exercise
may update its own evidence even when another exercise is missing or failed. Late
log corrections must recompute affected evidence and create an auditable replacement
decision; previous decisions are preserved.
