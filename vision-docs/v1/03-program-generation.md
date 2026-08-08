# Generating the program
From the above information we should be able to generate a program, with a gated check first:

Since v1.0 implements general strength training using the exact barbell, dumbbell,
pull-up, dip, and bodyweight-core exercises in `08-exercise-catalog.md`, the program
should still be structured as recommended by OG2:
1. Warmup
2. Skill Work
3. Strength Work
4. Optional Endurance / Conditioning
5. Rehab / Isolation / Flexibility/ Cooldown

but 2, 4, and 5 will be optional, and we will not prescribe 1. Perhaps that comes in a later version but for now we are simply programming for 3.

---
# Program Generation: v1 Pseudo-code

## Gates
There are individual Goal checks and general GoalSet checks/gates.
```
generate_program(athlete):
  require athlete has completed onboarding
  require athlete accepts the safety / clean-form disclaimer
  require athlete can train at least 2 sessions per week

  for goal in athlete.active_goals:
    require goal.adaptation == strength
    require goal.exercise revision is active in the selected v1 catalog release
    require goal.exercise.modality in [BARBELL, DUMBBELL,
                                       BODYWEIGHT_EXTERNAL_LOAD, BODYWEIGHT]
    require athlete has the equipment required for goal.exercise
    require goal has a measurable target and target date
    require athlete has a current capability for goal.exercise

  balance = assess_structural_balance(athlete.active_goals)
  if balance has uncovered movement categories and not athlete.overrides_balance_warning:
    return NEEDS_ATHLETE_DECISION(balance.warning)

  if athlete reports pain, worsening injury, or inability to perform clean reps:
    return NOT_READY_FOR_PROGRAMMING
```

`assess_structural_balance` checks vertical push, vertical pull, horizontal push,
horizontal pull, knee-dominant legs, posterior chain, and core/compression. An
athlete may deliberately override a warning, but NovaFit must record the reason.

## Initial Program

### Session Structure

NovaFit chooses session frequency before choosing a split. For beginner and
intermediate athletes, the preferred v1 structure is three full-body sessions per
week, separated by about 48 hours. Use two full-body sessions when only two
trainable days are available.

Use four upper/lower sessions only when the athlete has four compatible training
days and three full-body sessions cannot fit the required priority-goal and
declared-goal work within the maximum session duration. Do not add sessions merely
because additional days are available. v1 does not generate body-part, push/pull/legs,
or five-or-more-session splits.

Discard any session structure that cannot be scheduled on the athlete's trainable
days, exceeds the maximum session duration, or cannot provide the required goal
work. If no supported structure is feasible, return `NEEDS_ATHLETE_DECISION` with
the limiting constraints. Exact 48-hour separation is preferred, not required;
select the best valid schedule and record the reason when it is impossible.

### Schedule Selection

`trainable_days` is a hard constraint. For two full-body sessions, NovaFit prefers
weekly cyclic gaps of 3 and 4 days; for three, it prefers 2, 2, and 3 days. It
enumerates every subset of the required size from `trainable_days`, ranks candidates
by the largest shortest gap, then the smallest deviation from the preferred gap
pattern, and breaks any remaining tie by Monday–Sunday order. If the athlete has
fewer trainable days than the selected session frequency, return
`NEEDS_ATHLETE_DECISION(INSUFFICIENT_TRAINABLE_DAYS)`.

When no candidate has the preferred pattern, NovaFit selects the best valid schedule
and records `RECOVERY_SPACING_COMPROMISED` with its actual cyclic gaps. This is a
warning, not a generation blocker. For a permitted four-session upper/lower
structure, NovaFit alternates upper and lower sessions where possible and applies
the same ranking to the gaps between sessions of the same half.

```
choose_schedule(trainable_days, session_structure):
  required_days = session_structure.sessions_per_week
  if count(trainable_days) < required_days:
    return NEEDS_ATHLETE_DECISION(INSUFFICIENT_TRAINABLE_DAYS)

  candidates = all day subsets of size required_days
  if session_structure is TWO_FULL_BODY:
    rank by descending min(cyclic_gaps), ascending deviation(cyclic_gaps, [3, 4]), weekday_order
  else if session_structure is THREE_FULL_BODY:
    rank by descending min(cyclic_gaps), ascending deviation(cyclic_gaps, [2, 2, 3]), weekday_order
  else if session_structure is FOUR_UPPER_LOWER:
    assign alternating upper/lower labels where possible
    rank by descending min(same_half_cyclic_gaps), weekday_order

  schedule = highest-ranked candidate
  if schedule does not meet its preferred gap pattern:
    record RECOVERY_SPACING_COMPROMISED(actual_gaps(schedule))
  return schedule
```

### Initial Load

For a weighted exercise, NovaFit derives a single exercise-specific `reference_1RM`
from the athlete's current capability. A true 1RM is used directly. For a low-rep
assessment or an assessment with reported RIR, use the canonical estimated-1RM
calculation in `04-progression-policies.md`; an RIR-adjusted result uses completed
reps plus reported RIR. The assessment must be for the prescribed exercise. v1 does
not infer a starting load from a different exercise or an unassessed capability.

NovaFit chooses the initial prescribed rep target, selects the corresponding
conservative percentage below, multiplies it by `reference_1RM`, and rounds the
result down to the equipment's next valid load tier. Load conventions, athlete
inventory, and the default `5 lb` barbell and `5 lb`-per-hand dumbbell increments are
defined in `08-exercise-catalog.md`. These percentages are intended to start at
roughly 1–2 RIR; rounding must never increase the load.

| Initial rep target | Starting load |
| --- | --- |
| 3 | 87.5% of reference 1RM |
| 4 | 85% |
| 5 | 82.5% |
| 6 | 80% |
| 7 | 77.5% |
| 8 | 75% |

Any future calisthenics-specific programming policy may allow a goal exercise to
use a capability from an earlier, declared progression. Capability and loading must
then be progression-specific. That exception is outside v1.

Pike compressions and GHD sit-ups use their exact rep assessment and the catalog's
rep-only capability branch. They do not enter the estimated-1RM calculation or
receive external load in v1. Their prescription progresses through reps within the
declared program envelope; reaching that boundary ends the program for retesting
and regeneration rather than silently introducing load.

### Initial Prescription and Paired Sets

Every exercise selected for an initial strength program starts as a **primary**
compound exercise. `Primary` describes its treatment in this program, not an
inherent property of the exercise: several exercises may be primary in one session.
Goal priority determines exercise order; it does not demote lower-priority or
other declared goal work.

A primary prescription is three work sets, 3–8 reps, approximately 1–2 RIR,
3–5 minutes of effective rest, and `10X0` tempo. Beginners normally use a 5–8-rep
target and 2 RIR; choose the lower end of the range only when it serves the
athlete's goal. NovaFit does not generate isolation work in v1.

When primary work would otherwise exceed the athlete's maximum session duration,
NovaFit may use a paired set of two compatible exercises. A paired set alternates
the exercises with enough rest for each movement; it is not a conditioning circuit.
For example: perform A, rest 90 seconds, perform B, rest 90 seconds, then repeat.
Pair only exercises whose movement demands do not materially overlap, and never
pair an exercise when doing so would compromise its required rest or performance.

`Secondary` is not used during initial generation. It is a progression revision
that reduces a previously primary exercise from three to two work sets after
observed performance evidence indicates that the program's current demand is
excessive. Its other prescription variables carry over unchanged unless separately
changed by the normal progression policy. NovaFit must retain the reason and the
revision that made this change.

### Goal Exercise Selection

If an active goal has a current capability for its exercise, NovaFit selects that
exact exercise. It does not use form, movement-quality, or agent preference to
choose a substitute. A missing exact capability blocks program generation. Multiple
goals that select the same exercise produce one program exercise with links to all
of those goals; its ordering uses the highest priority.

### Weekly Goal Allocation

NovaFit gives every distinct declared goal exercise one primary weekly exposure.
An exercise appears no more than once in a session and no more times per week than
the selected session count. A primary exposure normally has three work sets; target
at least 15 total reps for an individual exercise when practical.

After assigning every baseline exposure, NovaFit assigns remaining feasible exercise
slots in priority rounds. It gives one additional exposure to every eligible exercise
in the highest-priority tier before considering the next tier, then starts another
round until no feasible slot remains. An exercise is eligible only when it has not
reached the selected session count and can be placed in a session within its duration
limit. Resolve ties by the exercise's canonical identifier. A session normally
contains no more than six main goal exercises.

Distinct goal exercises remain distinct even when they share a movement category.
Their prescribed work is accumulated and recorded under that category, but category
overlap never replaces, merges, or adds an exercise. Within each session, schedule
the highest-priority assigned exercise first; then schedule the remaining exercises
by priority, using compatible paired sets where applicable. If even the baseline
exposure for every declared goal cannot fit, return `NEEDS_ATHLETE_DECISION` rather
than silently dropping a goal.

### Volume Feasibility

OG2's strength-volume guidance is a soft planning target: roughly 25–50 weekly reps
for each represented push, pull, or leg category, and at least 15 total reps for an
individual exercise when practical. A category not represented by a declared goal is
not assigned work, including when the athlete has approved a structural-balance
override.

The baseline weekly exposure for every declared goal is the hard generation
requirement. NovaFit first uses compatible paired sets and, where the athlete has
four compatible days, the permitted upper/lower structure to fit that baseline. It
does not reduce initial primary work to secondary work to meet time. If the baseline
fits but a soft volume target or additional priority exposure does not, generate the
baseline program and record `VOLUME_TARGET_UNMET` with the category, target,
prescribed volume, and shortfall. If the baseline does not fit, return
`NEEDS_ATHLETE_DECISION` with the shortfall and the available choices: increase the
session-duration limit, add compatible training days, or pause/remove declared goals.

### Initial Mesocycle Duration

For beginner and intermediate athletes, an initial program has an eight-week maximum
duration rather than a preplanned four- or six-week endpoint. Their progression,
deload, and retest decisions are governed by the evidence policy in
`04-progression-policies.md`; broad stall, broad regression, safety, or goal
completion may end the program earlier. At the eight-week cap, complete the final
deload/retest and regenerate the program even if progress is continuing.

An advanced program must declare a planned mesocycle duration of 4–8 weeks, its
internal phases, and deload timing. Performance evidence may still trigger an early
intervention. v1 must not generate an advanced program without that declared plan.

```
choose_initial_mesocycle(athlete):
  if athlete.training_level in [BEGINNER, INTERMEDIATE]:
    return Mesocycle(max_weeks=8, progression_policy=EVIDENCE_LED)
  if athlete.training_level == ADVANCED:
    require athlete or policy supplies declared_plan(duration_weeks=4_to_8,
                                                      phases, deload_timing)
    return declared_plan
```

```
build_initial_program(athlete):
  sessions = choose_sessions(athlete)
  if sessions is NEEDS_ATHLETE_DECISION:
    return sessions
  schedule = choose_schedule(athlete.trainable_days, sessions.structure)
  if schedule is NEEDS_ATHLETE_DECISION:
    return schedule

  exercises = []
  for goal in athlete.active_goals, ordered by priority:
    exercises += goal.exercise  # exact capability is required by the gate

  merge duplicate exercises, retaining every linked goal and the highest priority

  for exercise in exercises:
    prescription = primary_strength_prescription(exercise, athlete.training_level)
    if exercise supports LOAD_FOR_REPS or EXTERNAL_LOAD_FOR_REPS:
      reference_1RM = capability_to_reference_1RM(exercise, athlete.capabilities)
      starting_load = round_down_to_valid_equipment_tier(
        reference_1RM * initial_load_percentage(prescription.rep_target)
      )
      # Never round upward or prescribe failure.
    else if exercise supports BODYWEIGHT_REPS:
      reference_max_reps = capability.completed_reps + capability.reported_RIR
      prescription = rep_only_strength_prescription(reference_max_reps,
                                                    program_policy_version)
    # Every branch requires an exact-exercise capability.

  weekly_allocation = assign_one_feasible_exposure_to_each(
    exercises, sessions, allow_compatible_paired_sets=true
  )
  if weekly_allocation is incomplete:
    return NEEDS_ATHLETE_DECISION(
      BASELINE_GOAL_WORK_DOES_NOT_FIT,
      options=[INCREASE_SESSION_DURATION, ADD_COMPATIBLE_DAYS, PAUSE_OR_REMOVE_GOALS]
    )
  while a feasible exercise slot remains:
    for priority_tier in priority_order:
      for exercise in priority_tier, ordered by canonical_exercise_id:
        assign one additional exposure if exercise is eligible
        # Fewer exposures than sessions, and fits its target session and duration,
        # including a compatible paired set when necessary.

  distribute exercises across sessions:
    place each exercise according to weekly_allocation
    put highest-priority assigned exercise first in each session
    use the selected full-body or upper/lower structure
    keep no more than 6 main goal exercises in a session
    keep at least 15 total reps for an individual exercise when practical
    if a session exceeds athlete.max_session_duration_minutes:
      add compatible paired sets without reducing each exercise's effective rest
    if a session still exceeds athlete.max_session_duration_minutes:
      return NEEDS_ATHLETE_DECISION(
        BASELINE_GOAL_WORK_DOES_NOT_FIT,
        options=[INCREASE_SESSION_DURATION, ADD_COMPATIBLE_DAYS, PAUSE_OR_REMOVE_GOALS]
      )

  volume_shortfalls = compare_declared_goal_volume_to_soft_targets(sessions)
  if volume_shortfalls exist:
    record VOLUME_TARGET_UNMET(volume_shortfalls)

  mesocycle = choose_initial_mesocycle(athlete)
  completion_policy = declare_completion_policy(athlete.active_goals, mesocycle)
  mutation_envelope = declare_v1_mutation_envelope(exercises, schedule)
  return program(version, catalog_release_id, exercise_revision_ids,
                 athlete_equipment_version, mesocycle, sessions,
                 completion_policy, mutation_envelope)
```

Progression after the initial program is generated is defined in `04-progression-policies.md`.
