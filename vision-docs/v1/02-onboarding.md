# Onboarding
The onboarding should collect information in the following order:
1. Generic Athlete Info
2. Athlete Goals
3. Athlete Current Capabilities

## Athlete Info
```
Athlete {
  age: 25,
  height: 178_cm,
  weight: 75_kg,

  training_level: beginner, # dictates pace of progress, more thought here as this is nuanced

  equipment: ["pull-up bar", "rings", "barbell"],
  training_availability: {
    max_sessions_per_week: 5,
    max_session_duration_minutes: 120,
    trainable_days: ["Monday", "Wednesday", "Friday"]
  },
}
```

In v1, `trainable_days` is a hard availability constraint, not a soft preference.
NovaFit never schedules a workout on another day.

## Athlete Goals
The second onboarding task is to establish the athlete's goals. Every goal should be SMART: specific, measurable, achievable, relevant, and time-bound. Here are some examples:

Every goal names exactly one supported exercise from the exercise library. That
exercise reference is canonical: when the athlete has a current capability for it,
NovaFit programs that exact exercise and does not substitute a variation.
Onboarding cannot activate the goal until the athlete supplies that direct,
exercise-specific capability assessment.

```
Goal {
  adaptation: strength
  domain: calisthenics
  exercise: free-standing-headstand-push-ups
  success_measure: reps
  target_value: 5
  target_date: 2026-12-31
  priority: 1
  feasibility: # assessed by NovaFit analysis
  status: active
}

Goal {
  adaptation: strength
  domain: barbell
  exercise: back-squat
  success_measure: reps
  load: 315lbs
  target_value: 1
  target_date: 2026-12-31
  priority: 2
  feasibility: # assessed by NovaFit analysis
  status: active
}
```

## Athlete Current Capabilities

Each capability should come from one clean near-maximal performance rather than
accumulated work across multiple sets. Prefer a true 1RM when appropriate; otherwise,
use a near-maximal low-rep set or the goal's specific success measure.

NovaFit accepts the athlete's report that the assessment and later training reps use
good form. Assessing form, pain, or movement quality is outside v1; the athlete is
responsible for reporting capabilities, RPE, and RIR honestly and for stopping when
they cannot perform an exercise safely.

```
Capability {
  domain: barbell
  exercise: back-squat
  success_measure: reps
  value: 1
  load: 285
  RPE: 10
  RIR: 0
  assessment_type: true_1RM
  assessed_at: 2026-08-01
}

Capability {
  domain: calisthenics
  exercise: chest-to-wall-headstand-pushups
  success_measure: reps
  value: 3
  RPE: 10
  RIR: 0
  assessment_type: max_rep_test
  assessed_at: 2026-08-01
}
```

This should be enough information for onboarding. The purpose of NovaFit is not to help with pain management etc. ; if the athlete mentions pain, mobility restrictions, etc. they need to consult a professional and return to NovaFit only when they can perform exercises safely and correctly with clean form. NovaFit is intended as program management, assuming that the athlete trains and performs exercises properly and is not cheating themselves about RPE, RIR, technique etc. This should be a general disclaimer.

---
# Onboarding Validation/Checks

## Individual Goal Validatiom
Individual Goals should be SMART:
- Specificity is satisfied by picking a specific exercise
- Measurability is satisfied by picking a target weight and rep count
- Attainability will have to be loosely judged. Perhaps Hermes can make a loose first binary guess. Feasibility/Attainability of goals will be continously tracked as Athlete progresses in relation to deadline
- Relevance is satisfied by picking specific exercise
- Time-bound is satisfied by picing a deadline

## GoalSet Validation

NovaFit evaluates the declared goals for structural balance: vertical push, vertical
pull, horizontal push, horizontal pull, knee-dominant legs, posterior chain, and
core/compression. Missing coverage raises a warning. The athlete must either add a
goal or explicitly override the warning with a reason—for example, because rock
climbing supplies pulling outside NovaFit. An approved override is accepted as the
athlete's responsibility; v1 does not model external training or add compensating
exercises to the program.
---

# Method of Onboarding
There are two ways that I envision the onboarding stage going:
1. via Chat Interface via Hermes
2. via Frontend Form

## Via Chat with Hermes
For the Chat route, the Athlete/User starts the onboarding process and is asked to provide `AthleteInfo`, `AthleteGoals`, and `CurrentCapabilities`. They provide this information in natural language. Hermes uses MCP to validate that information passes Onboarding Gates/Checks. Loop is finished once gates/checks pass.

## Frontend Form
For the frontend form, when onboarding starts, a frontend form is spun up on the computer, and the user fills out the information in the form. Everything has to pass validation. Upon success, onboarding is complete.

I am unclear on which onboarding route is easier. I am leaning towards via Chat interface.
