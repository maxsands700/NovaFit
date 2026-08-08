# NovaFit v1: Open Decisions Before Full Implementation

## Purpose

The existing vision documents provide enough direction to begin the foundation,
architecture, persistence, and core domain-model work. They do not yet define every
observable coaching and product behavior needed to build a deterministic NovaFit v1.

This document records those open decisions without changing the agreed vision. We
will review the decisions together, choose the best solution for each, and then
incorporate the approved behavior into the appropriate vision documents and
versioned policy configuration.

## Priority and Build Impact

| Decision | Build impact | Work that can proceed beforehand |
|---|---|---|
| 1. Initial program-generation policy | Blocks complete program generation | Foundation, domain models, persistence contracts |
| 2. Progression-policy implementation | Blocks audited next-workout decisions | Workout evidence models and decision interfaces |
| 3. Canonical v1 exercise catalog | Blocks reliable goal validation and exercise selection | Catalog schema and importer interfaces |
| 4. Workout evidence and safety semantics | Blocks final logging, progression, and Sheets contracts | Generic workout and revision models |
| 5. Supported athlete and goal scope | Affects onboarding and all coaching policies | Basic athlete, goal, and capability models |
| 6. Agent responsibility and goal feasibility | Blocks final onboarding validation | MCP/application boundary design |
| 7. Scheduling, Sheets submission, and operations | Blocks the final end-to-end delivery workflow | Core application workflow without Sheets |
| 8. Acceptance scenarios | Blocks confidence that the implementation matches the vision | Test infrastructure and individual unit tests |

---

## 1. Initial Program-Generation Policy

### Existing direction

NovaFit should normally create two or three full-body strength sessions, prioritize
the athlete's declared goals, validate structural balance at onboarding, stay within
time constraints, and prescribe approximately three sets of 3–8 reps at about one RIR. The starting
prescription should be derived from demonstrated capability and must not prescribe
failure.

### Approved decision: program split

NovaFit chooses session frequency before choosing a split. For beginner and
intermediate athletes, three full-body sessions are the preferred v1 structure;
two full-body sessions are used when only two trainable days are available. A
four-session upper/lower split is allowed only when four compatible days are
available and three full-body sessions cannot fit required priority-goal work within
the athlete's maximum session duration. Extra
available days alone do not justify extra sessions.

v1 does not generate body-part, push/pull/legs, or five-or-more-session splits.
Every candidate must fit the athlete's available days, duration limit, and safe
required work. If none does, program generation returns `NEEDS_ATHLETE_DECISION`
with the limiting constraints. Approximately 48-hour spacing is preferred, not a
hard requirement; the selected schedule must record when that preference cannot be
met.

### Why a decision is needed

The current pseudocode names the required operations but does not define a unique
result for a given athlete. Two valid implementations could select different
exercises, starting loads, rep ranges, schedules, and mesocycle lengths while both
appearing to satisfy the prose.

### Resolved checklist

- [x] **1.1 Starting load:** Normalize an exact-exercise capability to a reference
  1RM, apply the versioned 3–8-rep percentage table, and round down.
- [x] **1.2 Initial prescription:** Begin all selected work as primary compound
  work; use compatible paired sets for time, and reduce to secondary only through a
  recorded progression revision.
- [x] **1.3 Exact capability:** Select the exact exercise named by the goal; merge
  duplicate selections while retaining goal links and highest priority.
- [x] **1.4 Missing capability:** A current, direct, exact-exercise capability is
  required before a goal can be active or program generation can begin. A future
  calisthenics progression exception is out of v1 scope.
- [x] **1.5 Structural balance:** Validate the declared goal set. The athlete adds a
  goal or records an override; v1 generates only declared goals and accepts an
  approved override.
- [x] **1.6 Balance exercises:** Void in v1. NovaFit does not add exercises that are
  not declared goals.
- [x] **1.7 Weekly goal allocation:** Give every distinct goal exercise one weekly
  primary exposure, then award remaining feasible slots in priority rounds. Preserve
  distinct goals that share a movement category and return a decision if even the
  baseline does not fit.
- [x] **1.8 Volume feasibility:** Treat OG2 volume as a soft target. Generate and
  disclose a baseline program when it fits but misses that target; return an athlete
  decision only when even the baseline cannot fit.
- [x] **1.9 Schedule selection:** Treat trainable days as hard constraints and choose
  the deterministic two- or three-session schedule closest to OG2's recovery
  cadence; record a warning when that cadence cannot be met.
- [x] **1.10 Initial mesocycle:** Use an evidence-led eight-week maximum for beginner
  and intermediate programs; advanced programs require a declared 4–8-week planned
  mesocycle. Every program may end earlier for defined evidence or completion events.
- [x] **1.11 Completion and mutation:** Declare normal and early program-end reasons,
  retest/deload policy, and the exact mutable versus fixed fields. Revisions are
  bounded and evidence-backed; structural changes end the program and regenerate it.

All questions in this initial program-generation policy are resolved.

### Decision complete when

- The same versioned inputs always produce the same program.
- Every selection and calculation has a stable reason code.
- At least one exact worked example exists for a beginner and an intermediate.
- Conflicts among goals, balance, volume, duration, and availability have explicit
  precedence rules.

---

## 2. Progression-Policy Implementation

### Existing direction

The documents define beginner and intermediate double progression, exercise-level
pass/fail behavior, regression of one variable, program-level stall and regression
checks, deloads, retesting, immutable revisions, and a future-facing continuous
evidence model.

### Why a decision is needed

The discrete heuristics and continuous model do not yet form one executable policy.
The continuous model says that the training levels should not be rigid switches,
but its weights, thresholds, comparison windows, confidence calculation, recency
decay, and hysteresis are unspecified.

### Questions to resolve

1. Will the first policy version use the explicit discrete heuristics, a fully
   specified continuous model, or a deliberately staged combination of both?
2. What makes two exposures comparable?
3. How are completion, RIR delta, RPE, load/reps trend, estimated strength, failed
   reps, and technique combined?
4. What values define sufficient confidence and high or low evidence?
5. How many past exposures are considered, and how does evidence decay with time?
6. What exact hysteresis rule prevents progress/regress oscillation?
7. How and when does an exercise move between beginner, intermediate, and advanced
   progression behavior?
8. What is the exact definition of a primary exercise, broad stall, most exercises,
   and a meaningful RPE rise?
9. Does one failed exposure followed by another always cause regression, or can the
   continuous evidence state override that heuristic?
10. What exact prescription is used for a deload, and what counts toward the deload
    limit?
11. When does goal achievement trigger a retest, program completion, or immediate
    program regeneration?
12. What happens when RPE and RIR are absent or internally inconsistent?

### Decision complete when

- One authoritative algorithm is designated for the initial policy version.
- Every numeric parameter is stored in immutable, versioned configuration.
- All outcomes resolve to explicit actions and reason codes.
- Golden scenarios cover pass, easy pass, hard pass, isolated failure, repeated
  failure, broad regression, deload, retest, goal completion, and policy boundaries.

---

## 3. Canonical v1 Exercise Catalog

### Existing direction

V1 supports general weighted strength using barbell, dumbbell, and
weighted-bodyweight exercises. Program generation depends on a supported exercise
library containing modality, equipment, movement category, measures, technique
guidance, and progression information.

The supplied `data/OG2/novafit.sqlite3` is a useful source dataset, but it is not yet
explicitly designated as the canonical application catalog. It is primarily a broad
calisthenics catalog and does not expose every field required by the v1 contracts.

### Questions to resolve

1. What exact exercises are supported by v1?
2. Is `data/OG2/novafit.sqlite3` an authoritative source, an import fixture, or
   research data used to create a separate versioned catalog?
3. What is the canonical identifier for each exercise and each equipment item?
4. Which modality and movement categories apply to each exercise?
5. Which goal success measures and capability assessment types does each exercise
   support?
6. Is load represented as total load, external load, added load, or per-hand load?
7. What load increments are available, and are they athlete/equipment-specific?
8. What safety notes, technique cues, default tempo, and time estimate belong to each
   exercise?
9. How is a catalog version frozen and referenced by historical programs?

### Decision complete when

- A machine-readable, versioned v1 catalog is designated as authoritative.
- Every supported exercise satisfies one validated schema.
- Load conventions and supported measures are unambiguous.
- Program-generation and progression policies can reference stable catalog IDs and
  permitted increments without hard-coded exceptions.

---

## 4. Workout Evidence and Safety Semantics

### Existing direction

Progression evaluates prescribed and completed sets, reps, load, RPE/RIR, failed
reps, technique, and pain. Google Sheets currently allows the athlete to enter only
completed reps, RPE, RIR, failed reps, and pain; technique notes are read-only
NovaFit cues.

### Why a decision is needed

The progression contract requires evidence the current athlete-input contract cannot
always express. The documents also differ on whether pain stops only the affected
exercise or stops the entire program.

### Questions to resolve

1. Must the athlete always use the prescribed load, or must actual completed load be
   loggable per set?
2. How does the athlete report explicit technique breakdown?
3. Is `Failed Reps` a count, a per-set boolean, or an exercise-level signal?
4. How are skipped exercises, skipped sets, and workouts abandoned midway represented?
5. Which skip reasons are performance-related, pain-related, or non-comparable?
6. Are RPE and RIR both requested, or should the interface prefer one and treat the
   other as optional corroborating evidence?
7. What validation applies when RPE and RIR contradict each other?
8. When pain is reported, does NovaFit stop the exercise, hold the next workout, end
   the entire program, or choose among those actions using a declared rule?
9. Can an athlete later indicate that a pain entry was accidental, and how does that
   correction affect prior decisions?
10. What makes a partial workout complete enough to submit and process?

### Decision complete when

- Every progression input can be represented by the core workout schema.
- The non-Sheets and Sheets submission contracts have identical semantics.
- Missing, invalid, skipped, corrected, and contradictory evidence have explicit
  classifications.
- Pain behavior is consistent across onboarding, logging, progression, and the
  athlete-facing message.

---

## 5. Supported Athlete and Goal Scope

### Existing direction

Onboarding records an overall training level and goals from potentially different
domains. Program generation limits v1 to weighted-strength modalities, while
progression level is later intended to become exercise-specific. Detailed advanced
periodization is outside the first progression-policy version.

### Questions to resolve

1. Does v1 support beginners and intermediates only, or are advanced athletes
   supported through a limited fallback policy?
2. Is the onboarding `training_level` self-reported, assessed by NovaFit, or inferred
   per exercise from capability and training history?
3. What exact evidence initially classifies a lift as beginner, intermediate, or
   advanced?
4. Are unsupported goals rejected, stored as inactive future goals, or accepted but
   excluded from the current program?
5. How many active goals may an athlete have, and must priorities be unique?
6. What capability freshness window applies before NovaFit requires reassessment?
7. Are minors supported, and are there any lower/upper age restrictions or additional
   consent requirements?

### Decision complete when

- V1 inclusion and rejection rules can be expressed as deterministic validation.
- The relationship between overall training level and exercise-specific stage is
  defined.
- Unsupported goals and advanced athletes receive an explicit, safe product outcome.
- Structural balance is applied at one clearly identified boundary.

---

## 6. Agent Responsibility and Goal Feasibility

### Existing direction

Hermes gathers natural-language input, invokes narrow MCP tools, coordinates the
workflow, and explains deterministic decisions. Python owns validation, calculations,
policy execution, and persistence. The onboarding document currently suggests that
Hermes might make a loose binary judgment about goal attainability.

### Why a decision is needed

Allowing the agent to determine whether a goal is feasible would make a coaching gate
non-deterministic and could conflict with the architectural rule that the agent is an
interface rather than the coach.

### Questions to resolve

1. Is feasibility a hard onboarding gate, a non-blocking warning, or an informational
   estimate?
2. What deterministic inputs and formula, if any, determine initial feasibility?
3. Should Hermes only explain the application's result, or may it provide clearly
   labeled, non-authoritative commentary?
4. How often is feasibility recomputed as capabilities change and the deadline
   approaches?
5. What happens when the goal is measurable but NovaFit lacks sufficient evidence to
   judge feasibility?
6. Can an athlete proceed after an infeasibility warning, and must the override reason
   be recorded?

### Decision complete when

- The authoritative feasibility result comes from a versioned application policy or
  feasibility is explicitly removed as a hard gate.
- Unknown feasibility is distinct from feasible and infeasible.
- Hermes cannot silently introduce coaching decisions that are absent from the audit
  record.

---

## 7. Scheduling, Google Sheets Submission, and Operations

### Existing direction

NovaFit initially runs locally on a dedicated macOS user. Each athlete has one
workbook containing the currently actionable workout. NovaFit publishes a workout,
imports a completed snapshot idempotently, evaluates it, and publishes the next one.

### Questions to resolve

1. What athlete timezone and preferred session time must onboarding collect?
2. What happens when a workout is early, late, missed, skipped, or completed on a
   different day?
3. Can athletes reschedule, and which schedule changes require program regeneration?
4. How is the `Submit Workout Completion` action implemented: polling, an Apps Script,
   an MCP/chat action, a checkbox/state cell, or another mechanism?
5. How does the system prevent a new publication from overwriting an unprocessed or
   concurrently edited workout?
6. What exact hidden-sheet schema, hash algorithm, and validation contract are used?
7. How does an athlete request a correction after the workbook has already advanced
   to the next workout?
8. Is v1 single-athlete, multiple athletes under one operator, or genuinely
   multi-user?
9. What authentication or authorization protects FastAPI and MCP entry points?
10. What are the backup, restore, retention, deletion, log-redaction, and credential
    rotation requirements?
11. How are failed jobs, exhausted retries, and workouts awaiting manual intervention
    surfaced to the operator and athlete?

### Decision complete when

- The athlete can complete, submit, correct, miss, and reschedule a workout through
  explicit workflows.
- Publication/import concurrency and retry behavior are deterministic.
- The deployment has a minimal security, backup, and failure-recovery runbook.
- The Sheets submission mechanism is proven feasible with a small integration spike.

---

## 8. Acceptance Scenarios and Definition of Done

### Why a decision is needed

The vision contains examples and pseudocode, but no complete set of input/output
fixtures that establishes whether the finished coaching engine matches the intended
behavior. Exact scenarios are the fastest way to expose and resolve hidden policy
choices.

### Required golden scenarios

At minimum, define exact expected outputs for:

1. A beginner with three training days and one primary strength goal.
2. A beginner with only two available days.
3. An intermediate with multiple prioritized goals.
4. An athlete whose goals omit one or more structural-balance categories.
5. A goal lacking a current exact-exercise capability.
6. Successful double progression through the top of a rep range and a load increase.
7. An easy pass, pass, hard pass, first failure, and repeated failure.
8. Missing RPE/RIR with otherwise complete evidence.
9. A skipped exercise, partial workout, failed rep, technique breakdown, and pain
   report.
10. An isolated regression that does not deload the whole program.
11. Broad regression that does trigger a deload.
12. Capability retesting and program regeneration.
13. Goal achievement and normal program completion.
14. An idempotent Sheet resubmission and a later corrected submission.
15. A publication/import failure followed by a safe retry.

### Decision complete when

- Every scenario contains exact onboarding inputs, policy/catalog versions, expected
  program or decision output, and reason codes.
- The scenarios can become automated unit, integration, and end-to-end tests without
  developers inventing missing expected behavior.
- V1 has an explicit definition of done covering the complete athlete loop.

---

## Recommended Discussion Order

Resolve the decisions in this order because later answers depend on earlier ones:

1. Supported athlete and goal scope
2. Canonical v1 exercise catalog
3. Initial program-generation policy
4. Workout evidence and safety semantics
5. Progression-policy implementation
6. Agent responsibility and goal feasibility
7. Scheduling, Google Sheets submission, and operations
8. Acceptance scenarios and final v1 definition of done

## Decision Log

Record approved decisions here during discussion. After approval, move the normative
language into the relevant vision document and retain a short reference in this log.

| Date | Decision | Status | Vision document updated |
|---|---|---|---|
|  |  | Open |  |
