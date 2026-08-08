# Development Roadmap

Build NovaFit as a sequence of complete, reviewable modules. Finish and test each
phase before starting the next.

```text
Foundation -> Exercise Catalog -> Onboarding -> Program Generation
-> Workout Logging -> Progression -> Complete Loop -> API/MCP -> Google Sheets
```

## Development Rules

- Implement domain behavior, application use cases, persistence, and tests together.
- Add only the tables and abstractions required by the current phase.
- Keep coaching logic independent of FastAPI, MCP, SQLAlchemy, and external APIs.
- End every phase with passing tests and a runnable demonstration of its flow.
- Do not create empty future modules.

## 0. Foundation

- Create the `src/novafit` package structure.
- Configure dependencies, application settings, logging, and test tooling.
- Configure SQLite, SQLAlchemy, and Alembic migrations.
- Add shared IDs, units, clock, errors, and transaction boundaries.
- Verify the application starts, migrations run, and tests pass.

## 1. Exercise Catalog

- Model the UUID-based exercises, revisions, catalog releases, modalities, movement
  categories, equipment, load conventions, and supported measures specified in
  `08-exercise-catalog.md`.
- Create the source-controlled v1 catalog release and load it idempotently into the
  new application database. Do not import, modify, or depend on
  `data/OG2/novafit.sqlite3`.
- Add catalog queries and persistence.
- Demonstrate stable IDs across database rebuilds, catalog validation, exercise
  lookup, historical revision resolution, and goal/exercise support validation.

## 2. Onboarding

- Model athletes, availability, equipment, goals, capabilities, consent, and overrides.
- Implement individual-goal and structural-balance validation.
- Persist and retrieve onboarding state.
- Demonstrate completing valid onboarding through an application use case.

## 3. Program Generation

- Implement the versioned weighted-strength programming policy.
- Model programs, training tracks, sessions, prescriptions, revisions, and mutation
  envelopes.
- Persist the generated program and its complete decision inputs and policy version.
- Demonstrate generating an auditable initial program from onboarded athlete data.

## 4. Scheduling and Workout Logging

- Model scheduled sessions, prescribed sets, completed sets, and completion revisions.
- Implement workout scheduling, completion submission, and correction use cases.
- Preserve immutable workout history and stable identities.
- Demonstrate recording a completed workout without Google Sheets.

## 5. Progression Policies

- Implement the rules in `04-progression-policies.md` as versioned policy code and
  configuration.
- Build decision snapshots, reason codes, confidence/evidence records, and program-end
  reports.
- Persist each decision with its exact inputs, output, policy version, and explanation.
- Demonstrate producing an audited next-exposure decision from a completed workout.

## 6. Complete Training Loop

- Connect workout completion to progression evaluation.
- Create the next prescription revision and scheduled session.
- Handle program stopping conditions and handoff reports to program generation.
- Demonstrate several simulated workout cycles end to end.

## 7. FastAPI and MCP Entry Points

- Expose the existing application commands and queries through thin FastAPI routes.
- Add narrow MCP tools for onboarding, program generation, workout completion, and
  retrieving decisions/prescriptions.
- Ensure entry points contain no coaching or persistence logic.
- Demonstrate the complete loop through both application interfaces.

## 8. Google Sheets

- Implement the publication and import contracts in `05-google-sheets.md`.
- Add protected identities, validation, idempotency, correction handling, and retryable
  jobs.
- Connect Sheet import to the existing workout-completion use case.
- Demonstrate the complete training loop through the athlete workbook.

## After V1

Add future capabilities only after their use cases and policies are specified:

1. Google Calendar scheduling adapter.
2. WHOOP and provider-neutral recovery/sleep observations.
3. Provider-neutral nutrition observations and tracking adapters.
4. Additional adaptation policies and hybrid-program coordination.
