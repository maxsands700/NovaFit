# Architecture

## Recommendation

Build NovaFit as a **modular monolith with hexagonal boundaries**.

There should initially be one Python application, one database, and one deployable
system, with FastAPI, MCP, scheduled jobs, and a future CLI acting as entry points to
the same application. Inside it, organize code by coaching capability and enforce a
one-way dependency rule:

```text
entry points and external adapters -> application use cases -> domain
```

The domain must not import FastAPI, MCP/Hermes, SQLAlchemy, Google APIs, WHOOP SDKs,
or any nutrition-provider SDK. Those technologies should be replaceable details at
the edge of the system.

This is preferable to microservices now. NovaFit's programming, workout logging,
and progression operations need consistent transactions and will evolve together.
Splitting them across services would add network failure modes, distributed data,
and operational cost without creating a useful product boundary. Well-defined
modules can be extracted later if scale, team ownership, or independent deployment
creates a concrete reason.

## Architectural Goals

The architecture should preserve these invariants:

1. **NovaFit is the source of truth.** Sheets, calendars, wearables, and nutrition
   apps are projections or evidence sources, not owners of NovaFit programs and
   decisions.
2. **Coaching behavior is deterministic, versioned, and auditable.** Given the same
   policy version and input snapshot, the engine should produce the same decision.
3. **External data never leaks into the core in vendor-specific shapes.** A WHOOP
   recovery score is translated at the boundary into provenance-rich observations;
   it is not passed through the programming engine as a WHOOP API object.
4. **Missing evidence is explicit.** Missing or stale sleep, nutrition, RPE, or RIR
   data remains unknown. It is never silently converted to a neutral or healthy
   value.
5. **Integrations are idempotent.** Retrying a Sheet import, calendar update, webhook,
   or background sync must not duplicate a workout, event, or observation.
6. **Safety rules outrank automation.** Pain and stop conditions cannot be overridden
   by readiness data, an agent, or an integration.
7. **The agent is an interface, not the coach.** Hermes gathers intent, invokes
   narrow use cases, and explains results. Python owns validation, calculations,
   policy execution, and persistence.

## System Shape

```text
                         NovaFit application

  FastAPI     MCP       scheduled jobs       future CLI/web UI
     \         |             |                    /
      +--------+------ entry points -------------+
                           |
                    application use cases
              (transactions, authorization, orchestration)
                           |
       +---------------- domain modules ----------------+
       | athletes | catalog | programming | progression |
       | workout log | scheduling | readiness | nutrition|
       +-------------------------------------------------+
                           |
                  ports owned by the use case
                           |
       +-------------------+-------------------------+
       | SQLAlchemy | Sheets | Calendar | WHOOP      |
       | email/etc. | nutrition providers | event jobs|
       +---------------- external adapters ----------+
```

The diagram is a dependency map, not a request sequence. Adapters implement ports
defined by the application/domain side. The core does not call a concrete Google or
WHOOP client.

## Proposed Repository Layout

Keep a single repository and a single installable Python package while there is one
team and one product deployment:

```text
NovaFit-parent/
├── pyproject.toml
├── README.md
├── .env.example                 # names only; never secrets
├── src/
│   └── novafit/
│       ├── shared/              # small, stable primitives only
│       │   ├── ids.py
│       │   ├── units.py
│       │   ├── clock.py
│       │   ├── events.py
│       │   └── errors.py
│       ├── athletes/
│       ├── exercise_catalog/
│       ├── programming/
│       ├── progression/
│       ├── workout_log/
│       ├── scheduling/          # planned sessions, not Google Calendar code
│       ├── readiness/           # future normalized recovery/sleep evidence
│       ├── nutrition/           # future normalized nutrition evidence
│       ├── application/
│       │   ├── commands/        # state-changing use cases
│       │   ├── queries/         # read use cases
│       │   ├── ports/           # DB and external-service Protocols
│       │   └── unit_of_work.py
│       ├── adapters/
│       │   ├── persistence/
│       │   │   └── sqlalchemy/
│       │   ├── google_sheets/
│       │   ├── google_calendar/ # future
│       │   ├── whoop/           # future
│       │   └── nutrition/       # future provider adapters
│       ├── entrypoints/
│       │   ├── api/             # FastAPI routes and API DTOs
│       │   ├── mcp/             # narrow Hermes tools
│       │   └── jobs/            # syncs, publication, retry workers
│       ├── bootstrap.py         # dependency wiring only
│       └── config.py
├── migrations/                  # Alembic migrations
├── policy-config/
│   ├── weighted_strength/       # immutable/versioned policy parameters
│   └── exercise_catalog/
├── tests/
│   ├── unit/                    # pure domain and policy tests
│   ├── integration/             # database and concrete adapters
│   ├── contract/                # ports and third-party payload fixtures
│   └── e2e/                     # a few complete athlete workflows
├── fixtures/                    # sanitized, non-secret test inputs
├── scripts/                     # thin operational/development commands
├── vision-docs/
└── docs/
    └── adr/                     # short architecture decision records
```

This is the target shape, not a requirement to create empty packages now. Add
`readiness`, `nutrition`, and concrete future adapters when their first use cases
are implemented. Recording the intended boundary is enough until then.

If a separately built web/mobile client is added, introduce `apps/web` or
`apps/mobile` at that point and generate a client contract from the API schema. Do
not move shared Python domain code into a language-neutral `packages` folder just in
case a client is later created.

### Inside a Capability Module

Each domain module owns its vocabulary and invariants. A module can start small and
split only when needed:

```text
programming/
├── domain.py          # Program, Prescription, ProgramRevision
├── policies.py        # policy interfaces and implementations
├── services.py        # domain operations spanning several entities
├── events.py
└── errors.py
```

Avoid repeating `controllers/services/models/repositories` inside every folder by
default. Create a file only when the behavior exists. Repository interfaces belong
with the application use case that needs them; SQLAlchemy implementations belong
under `adapters/persistence`.

## Module Responsibilities

| Module | Owns | Does not own |
|---|---|---|
| `athletes` | Athlete profile, availability, equipment, consent/overrides | Program generation |
| `exercise_catalog` | Exercise identity, modality, equipment, movement category, supported measures | Athlete prescriptions |
| `programming` | Goal validation, program construction, mutation envelopes, program versions | Importing Sheet cells |
| `progression` | Evidence evaluation and the next-exposure/program-level decision | Choosing an undeclared replacement program |
| `workout_log` | Scheduled prescription execution, completed sets, immutable corrections | Progression policy |
| `scheduling` | Planned session identity, local time, rescheduling intent and publication status | Google event payloads |
| `readiness` | Normalized sleep/recovery observations, freshness and provenance | WHOOP authentication or policy decisions |
| `nutrition` | Normalized nutrition observations, freshness and provenance | Vendor API objects or clinical advice |

Cross-module workflows belong in application use cases. For example,
`SubmitWorkout` validates and stores a completion through `workout_log`, invokes a
versioned `progression` policy, stores the complete decision record, and requests
publication of the next prescription. Neither a FastAPI route nor an MCP tool should
reimplement those steps.

## Classes and OOP

Use OOP selectively. Python classes are useful here, but making the whole system an
inheritance hierarchy would make future adaptations harder to change.

Use classes for:

- entities with identity and lifecycle, such as `Athlete`, `Goal`, `Program`,
  `PrescriptionRevision`, and `CompletedWorkout`;
- immutable value objects that enforce units or constraints, such as `Load`,
  `RepRange`, `RIR`, and `LocalSessionTime`;
- application use-case handlers that receive dependencies explicitly;
- stateful external clients and repository/unit-of-work implementations; and
- `typing.Protocol` contracts where multiple implementations are real or imminent.

Prefer pure functions for calculations and policy rules when the output depends only
on explicit inputs. Estimated strength, volume, comparable-exposure selection,
evidence scoring, and most progression decisions should be easy to test as ordinary
functions. A class is justified when it protects invariants, owns meaningful state,
or bundles an injected dependency—not merely because a noun or verb exists.

Use composition instead of inheritance for training types. Do not create a tree such
as `Program -> StrengthProgram -> CalisthenicsStrengthProgram`. Training plans will
eventually combine adaptations and domains, and hybrid training does not fit cleanly
into a single inheritance branch. Compose a program from tracks and versioned policy
objects instead:

```python
class ProgrammingPolicy(Protocol):
    policy_id: str
    version: str

    def supports(self, goal: Goal) -> bool: ...
    def build_tracks(self, context: ProgrammingContext) -> list[TrainingTrack]: ...


class ProgressionPolicy(Protocol):
    policy_id: str
    version: str

    def decide(self, snapshot: DecisionSnapshot) -> ProgressionDecision: ...
```

A registry can select policies by supported adaptation and domain. Weighted strength
is the first implementation. Later implementations can add hypertrophy,
calisthenics, endurance, or hybrid track coordination without adding scattered
`if goal.adaptation == ...` branches throughout the application. Keep programming
and progression contracts separate: generating a training track and adapting an
existing one are different responsibilities.

Avoid:

- `NovaFitManager`, `WorkoutService`, or other god objects;
- deep base-class hierarchies;
- a class for every function;
- static/global service locators;
- ORM objects traveling through the whole application; and
- generic `dict[str, Any]` domain models that postpone validation until runtime.

Use Pydantic for API/MCP/integration DTO validation. Use domain entities/value
objects for coaching behavior, and keep SQLAlchemy mappings in the persistence
adapter. For a simple record the Pydantic and domain shape may initially be the same;
do not create duplicate mapping layers until their responsibilities actually differ.

## Extending Training Adaptations

An adaptation is a versioned collection of capabilities, not a new application:

1. catalog metadata and supported goal measures;
2. onboarding/goal validation rules;
3. a programming policy that produces one or more `TrainingTrack` objects;
4. a mutation envelope and progression policy;
5. policy configuration and citations/provenance; and
6. contract tests demonstrating its allowed decisions.

`Program` is an aggregate of scheduled sessions and training tracks. A track states
its goal, movement/domain, prescription, policy identity, and permitted mutations.
This supports a future hybrid program by composing, for example, strength and
endurance tracks and then applying an explicit coordination policy for fatigue,
priority, and scheduling conflicts. Hybrid behavior must not emerge accidentally
from two policies independently editing the same week.

Policy configuration should be immutable once used. Store `policy_id`, version,
configuration checksum, exact input snapshot, output, and explanation with every
decision. A new threshold or use of recovery evidence creates a new policy version;
it does not rewrite the meaning of an old result.

## Adding External Integrations

Define a port around NovaFit's use case, not around a provider's API. A useful port
sounds like `CalendarPublisher.upsert_session(session)` or
`RecoverySource.fetch_observations(since)`, not `GoogleApiClient.execute(payload)`.
Concrete adapters translate between that port and the vendor.

### Google Sheets and Google Calendar

Google Sheets remains a workout publication/import adapter implementing the rules in
`05-google-sheets.md`. The application gives it a publication DTO and receives a
validated completion DTO; Sheet ranges and cell formatting never enter the domain.

The scheduling module should assign every planned session a stable internal ID.
Google Calendar stores a mapping between that ID and the provider event ID, making
create/update/retry idempotent. A prescription revision may update the event details
without changing the planned session's identity. Whether athlete calendar edits are
one-way display changes or accepted rescheduling commands must be an explicit product
decision; do not infer domain changes directly from arbitrary event fields.

### WHOOP and Recovery/Sleep Sources

Keep three representations separate:

1. the raw provider response, retained when permitted for debugging and replay;
2. normalized `SleepObservation` and `RecoveryObservation` records with source,
   observed interval, ingestion time, units, and provider algorithm/version when
   known; and
3. a `TrainingContextSnapshot` containing only the evidence an explicit policy is
   allowed to use for one decision.

V1 policies ignore these observations. A future recovery-aware policy must declare
which normalized fields it consumes, its freshness rules, how missing data behaves,
and how the signal contributed to the result. Vendor readiness scores should not
silently become medical facts or override pain/safety behavior.

### Nutrition Sources

Create provider-specific adapters only after choosing the first concrete use case.
They should translate into a provider-neutral model such as a timestamped daily
summary or nutrient observation, with units, provenance, completeness, and timezone.
The progression engine consumes a versioned context snapshot, not MyFitnessPal,
Cronometer, or another provider's schema. This permits manual logging or another
provider to implement the same port later.

### Integration Reliability

Use stable internal IDs, external-ID mappings, sync cursors, unique idempotency keys,
and webhook receipt records. Store OAuth tokens/credentials outside domain tables and
encrypt them at rest; never log or commit them. Rate limits, retries with backoff,
payload validation, and dead-letter/error visibility belong in adapters/jobs.

Start with direct in-process calls for the v1 workflow. Introduce named domain events
such as `WorkoutPublished`, `WorkoutCompleted`, `PrescriptionRevised`, and
`SessionRescheduled` so integrations can react without being embedded in coaching
logic. When losing an event would matter, persist it in an outbox in the same database
transaction and let a job deliver it. A network call must not be part of the database
transaction.

## Persistence and Data Ownership

- Use SQLite initially through SQLAlchemy, but add Alembic migrations from the first
  schema. Avoid SQLite-specific domain assumptions so PostgreSQL remains a deployment
  change rather than a rewrite.
- Generate stable application IDs (UUID/ULID) rather than exposing integer database
  row IDs to Sheets, calendars, API clients, or webhooks.
- Keep completed workouts, program/prescription revisions, and progression decisions
  append-only. A correction supersedes a prior record; it does not erase history.
- Store timestamps in UTC and preserve the athlete/session timezone needed to render
  local schedules correctly.
- Keep provider payloads and synchronization metadata separate from normalized
  observations. Retention and deletion can then be applied without corrupting the
  coaching record.
- Use a unit of work for each command so the state change, decision audit, and outbox
  events commit atomically.

Do not build one universal `metrics(key, value)` table for workouts, recovery,
nutrition, and every future signal. It looks flexible but discards units, invariants,
relationships, and useful constraints. Use typed domain records, allowing a JSON
column only for retained raw payloads or truly provider-specific metadata.

## Entry Points and Contracts

FastAPI routes, MCP tools, jobs, and a future UI must call the same command/query
handlers. Entry points may authenticate, validate/serialize a DTO, invoke one use
case, and translate its result. They may not query tables directly or implement
coaching policy.

Keep MCP tools narrow and task-oriented—for example `complete_onboarding`,
`generate_program`, `publish_next_workout`, and `submit_workout`—rather than exposing
arbitrary SQL, files, or generic entity mutation. Return structured decision reasons
so Hermes explains the recorded result instead of inventing a rationale.

Run background jobs through the same use cases and make each job safe to retry. Use
an injected `Clock` at decision boundaries so date-sensitive behavior and calendar
logic are deterministic in tests.

## Testing Strategy

1. **Unit tests:** policy examples and edge cases, with no database/network. Convert
   the progression examples in `04-progression-policies.md` into executable table
   tests and add property tests for hard invariants where useful.
2. **Integration tests:** SQLAlchemy repositories against SQLite and, before a
   PostgreSQL migration, PostgreSQL; Google/WHOOP/nutrition adapters against recorded,
   sanitized payloads and provider sandboxes where available.
3. **Contract tests:** every adapter must satisfy the port's shared behavior,
   including idempotency, pagination, missing fields, units, timezones, and stale
   evidence.
4. **End-to-end tests:** a small set covering onboarding -> program -> Sheet publish
   -> completion -> audited progression -> next publication.

Tests should assert decision output, reason codes, policy/configuration version, and
input provenance—not only a final prescribed load. External API responses in
fixtures must be sanitized and contain no credentials or athlete-identifying data.

## Practical Build Order

1. Create packaging, configuration, migrations, stable IDs/units, and test structure.
2. Implement athlete, goal, and exercise-catalog models plus onboarding use cases.
3. Implement weighted-strength program generation as the first policy.
4. Implement workout logging and the versioned progression policy with complete
   decision auditing.
5. Add FastAPI and MCP entry points over those same use cases.
6. Add the Google Sheets adapter and retry-safe publication/import jobs.
7. Add calendar, readiness, and nutrition modules only as their product use cases are
   specified; reuse the ports, normalized observations, snapshots, and audit model
   above.

This order delivers v1 without closing future doors. The important investment is not
empty abstractions: it is keeping coaching logic pure, decision inputs explicit,
module ownership clear, and external systems behind replaceable boundaries.
