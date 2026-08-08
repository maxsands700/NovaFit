# Generating the program
From the above information we should be able to generate a program, with a gated check first:

Since v1.0 implements general weighted strength training—barbell, dumbbell, and weighted-bodyweight exercises—the program should still be structured as recommended by OG2:
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
    require goal.exercise is supported by the v1 exercise library
    require goal.exercise.modality in [barbell, dumbbell, weighted_bodyweight]
    require athlete has the equipment required for goal.exercise
    require goal has a measurable target and target date
    require athlete has a current capability for the goal or a safe prerequisite

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
structural-balance work within the maximum session duration. Do not add sessions
merely because additional days are available. v1 does not generate body-part,
push/pull/legs, or five-or-more-session splits.

Discard any session structure that cannot be scheduled on the athlete's available
days, exceeds the maximum session duration, or cannot provide the required goal
and balance work safely. If no supported structure is feasible, return
`NEEDS_ATHLETE_DECISION` with the limiting constraints. Exact 48-hour separation
is preferred, not required; select the best valid schedule and record the reason
when it is impossible.

```
build_initial_program(athlete):
  sessions = choose_sessions(athlete)
  # Apply the Session Structure policy above.

  exercises = []
  for goal in athlete.active_goals, ordered by priority:
    exercises += select_goal_exercise(goal, athlete.capabilities, athlete.equipment)

  exercises += select_balance_exercises(exercises, balance)
  # Prefer the simplest exercise that directly serves the goal, can be performed
  # cleanly, and allows progressive overload.

  for exercise in exercises:
    prescription = default_strength_prescription(exercise, athlete.training_level)
    # Start from the athlete's demonstrated capability; do not prescribe failure.
    # Default: 3 sets, 3–8 reps, about 1 RIR, 3–5 minutes rest, 10X0 tempo.
    # Use a moderate rep range for beginners when additional practice is useful.

  distribute exercises across sessions:
    put highest-priority goal exercises first
    use the selected full-body or upper/lower structure
    keep weekly push and pull volume reasonably balanced
    target roughly 25–50 weekly reps per push, pull, and leg category
    keep at least 15 total reps for an individual exercise when practical
    keep the session within athlete.max_session_duration_minutes

  return program(version, mesocycle_length=4_to_8_weeks, sessions)
```

Progression after the initial program is generated is defined in `04-progression-policies.md`.
