# Canonical v1 Exercise Catalog

## Authority and Scope

The authoritative v1 catalog will be the source-controlled, machine-readable
catalog release at `policy-config/exercise_catalog/v1.0.0`. The application loads
that release into NovaFit's new application database through migrations and an
idempotent seed. A database rebuild must recreate the same catalog identities and
content; neither database-local row numbers nor display names are identities.

V1 contains exactly the following 21 general-strength exercises. Adding, removing,
renaming, or changing the policy metadata of an exercise requires a new catalog
release.

| Canonical exercise | Slug | Modality | Primary movement category | Laterality |
|---|---|---|---|---|
| Flat Barbell Bench Press | `flat-barbell-bench-press` | `BARBELL` | Horizontal push | Bilateral |
| Flat Dumbbell Bench Press | `flat-dumbbell-bench-press` | `DUMBBELL` | Horizontal push | Bilateral |
| Incline Barbell Bench Press | `incline-barbell-bench-press` | `BARBELL` | Horizontal push | Bilateral |
| Incline Dumbbell Bench Press | `incline-dumbbell-bench-press` | `DUMBBELL` | Horizontal push | Bilateral |
| Barbell Overhead Press | `barbell-overhead-press` | `BARBELL` | Vertical push | Bilateral |
| Dumbbell Shoulder Press | `dumbbell-shoulder-press` | `DUMBBELL` | Vertical push | Bilateral |
| Dip | `dip` | `BODYWEIGHT_EXTERNAL_LOAD` | Vertical push | Bilateral |
| Pull-up | `pull-up` | `BODYWEIGHT_EXTERNAL_LOAD` | Vertical pull | Bilateral |
| Bent-over Barbell Row | `bent-over-barbell-row` | `BARBELL` | Horizontal pull | Bilateral |
| Single-arm Dumbbell Row | `single-arm-dumbbell-row` | `DUMBBELL` | Horizontal pull | Unilateral |
| Barbell Back Squat | `barbell-back-squat` | `BARBELL` | Knee-dominant legs | Bilateral |
| Barbell Front Squat | `barbell-front-squat` | `BARBELL` | Knee-dominant legs | Bilateral |
| Barbell Bulgarian Split Squat | `barbell-bulgarian-split-squat` | `BARBELL` | Knee-dominant legs | Unilateral |
| Dumbbell Bulgarian Split Squat | `dumbbell-bulgarian-split-squat` | `DUMBBELL` | Knee-dominant legs | Unilateral |
| Dumbbell Goblet Squat | `dumbbell-goblet-squat` | `DUMBBELL` | Knee-dominant legs | Bilateral |
| Barbell Conventional Deadlift | `barbell-conventional-deadlift` | `BARBELL` | Posterior chain | Bilateral |
| Barbell Romanian Deadlift | `barbell-romanian-deadlift` | `BARBELL` | Posterior chain | Bilateral |
| Dumbbell Romanian Deadlift | `dumbbell-romanian-deadlift` | `DUMBBELL` | Posterior chain | Bilateral |
| Barbell Hip Thrust | `barbell-hip-thrust` | `BARBELL` | Posterior chain | Bilateral |
| Pike Compression | `pike-compression` | `BODYWEIGHT` | Core/compression | Bilateral |
| GHD Sit-up | `ghd-sit-up` | `BODYWEIGHT` | Core/compression | Bilateral |

`Pull-up` and `Dip` are the exercise identities whether performed at bodyweight or
with external load. Loading is prescription data, so the catalog must not create
separate "weighted" exercise identities. Planks, ab-wheel rollouts, chin-ups,
isolation work, and other squat, hinge, press, or row variations are outside the
initial v1 catalog.

Each exercise has exactly one primary structural-balance category. Additional tags
may describe hip hinge, posterior-chain contribution, unilateral loading, or setup,
but tags never cause goals to merge or replace one another.

## Identity and Schema

Every `Exercise`, `ExerciseRevision`, `EquipmentType`, athlete equipment record, and
`CatalogRelease` has an immutable UUID. For the initial catalog, UUIDv4 values are
generated once, written explicitly into the source-controlled seed, and reused in
every environment. Imports must never generate replacement IDs for known records.

An exercise slug is a unique, human-readable alternate key. It may be used for
lookup and seed review, but foreign keys and historical program references use the
UUID. Display names and slugs are versioned metadata rather than identity.

Every published exercise revision must validate at least these fields:

```
ExerciseRevision {
  exercise_revision_id: UUID
  exercise_id: UUID
  catalog_release_id: UUID
  slug: unique string
  display_name: string
  modality: BARBELL | DUMBBELL | BODYWEIGHT_EXTERNAL_LOAD | BODYWEIGHT
  primary_movement_category: enum
  movement_tags: set[enum]
  laterality: BILATERAL | UNILATERAL
  equipment_requirements: boolean groups[EquipmentType UUID]
  optional_loading_equipment: set[EquipmentType UUID]
  supported_goal_measures: nonempty set[enum]
  supported_assessment_types: nonempty set[enum]
  load_convention: enum
  execution_standard: string
  technique_cues: 1..3 strings
  common_faults: nonempty set[string]
  safety_note: string
  default_tempo: string
  setup_seconds: nonnegative integer
  side_transition_seconds: nonnegative integer
  equipment_transition_group: string
  status: ACTIVE | DEPRECATED
}
```

## Measures and Capability Assessments

Barbell and dumbbell exercises support `LOAD_FOR_REPS`. Their direct capability may
be a true 1RM or a clean, near-maximal low-rep set with load and RIR. Pull-ups and
dips support `EXTERNAL_LOAD_FOR_REPS` and `BODYWEIGHT_REPS`; their direct capability
may be a true external-load 1RM, a loaded low-rep set, or a clean near-maximal
bodyweight rep assessment.

Pike compressions and GHD sit-ups support `BODYWEIGHT_REPS`. Their direct capability
is one clean near-maximal rep assessment. They use a rep-only capability branch,
not an estimated-load branch:

```
reference_max_reps = completed_reps + reported_RIR
```

RPE and RIR are mandatory and must pass the consistency rule in
`04-progression-policies.md` for every assessment. Every assessment is tied to the
exact exercise UUID; v1 never transfers capability between variations.

## Load Conventions and Increments

NovaFit stores load internally as decimal kilograms, renders it in the athlete's
preferred unit, and preserves the originally submitted value and unit for audit.
The following conventions determine what a displayed prescription means:

| Exercise loading | Stored/displayed prescription load |
|---|---|
| Barbell | Total external load, including the bar |
| Paired dumbbells, one dumbbell per hand | Load of one dumbbell, explicitly labelled `per hand` |
| Single-arm dumbbell row | Load of the working-hand dumbbell |
| Goblet squat | Total load of the one dumbbell held |
| Pull-up or dip | External added load; bodyweight is stored separately with the exposure |
| Pike compression or GHD sit-up | No external load in v1 |

Actual athlete/equipment inventory determines valid load tiers. When detailed
inventory has not supplied a smaller supported tier, the v1 defaults are:

- barbell: `5 lb` total;
- paired dumbbells: `5 lb` per hand, or `10 lb` aggregate external load; and
- a one-dumbbell exercise: `5 lb` on the working or held dumbbell.

For a barbell with plate inventory, the next tier is the smallest supported total
load and normally equals twice the smallest available plate. Fixed dumbbells advance
to the next available pair; adjustable dumbbells advance to the next supported
setting. Pull-up and dip external load advances to the next configuration supported
by the athlete's attachment and available weight. The program records its permitted
load tiers rather than hard-coding exercise-name exceptions.

At minimum, the equipment catalog must distinguish barbell, plates, rack, flat
bench, adjustable bench, dumbbells, pull-up bar, dip station, external-load
attachment, GHD, and floor space. An athlete equipment record references the
equipment-type UUID and records the actual bars, plate denominations, fixed
dumbbell pairs or adjustable settings, and external-load configurations available.
Equipment requirement groups must support alternatives such as a flat bench or an
adjustable bench set flat without falsely requiring both.

For pull-ups and dips, estimated strength uses total system load
(`assessment bodyweight + external load`) while the athlete-facing result continues
to display external load. Bodyweight must therefore be snapshotted with each such
assessment or completed set.

## Guidance, Tempo, and Time Estimation

Catalog guidance is read-only instruction. It helps the athlete reproduce the
declared exercise standard; NovaFit does not score form or use perceived form to
select a different exercise. The athlete remains responsible for using good form
and stopping when an exercise cannot be performed safely.

The default tempo is `10X0` for loaded compounds, pull-ups, and dips, and `20X0` for
pike compressions and GHD sit-ups. Unilateral prescriptions are performed and
logged per side. `X` is rendered as an intentionally fast concentric and uses the
catalog's configured estimate when computing duration.

Session feasibility derives time instead of assigning one opaque duration to an
exercise:

```
set_time = setup_seconds
         + reps * estimated_seconds_per_rep(default_tempo)
         + side_transition_seconds_if_unilateral

exercise_block_time = sum(set_time for prescribed sets)
                    + effective_rest_between_sets
                    + equipment_transition_time
```

## Versioning and Historical Programs

A published `CatalogRelease` has an immutable UUID, semantic version, publication
timestamp, and checksum of its canonical serialized content. A stable `Exercise`
UUID spans releases; an immutable `ExerciseRevision` UUID identifies the complete
policy definition in one release. Published records are never updated or deleted.
A correction creates a new release and revision, while removal marks an exercise
deprecated for new programs.

Every program stores the catalog-release UUID, each exact exercise UUID and
exercise-revision UUID, and the athlete-equipment inventory version used to generate
it. Historical programs continue to resolve their frozen revisions after a newer
catalog is published.

## Legacy OG2 Database Boundary

`data/OG2/novafit.sqlite3` is immutable research material from an earlier,
calisthenics-focused NovaFit attempt. It is not an authoritative catalog, import
fixture, migration source, seed, test dependency, or runtime dependency. No v1
application or migration code may modify it or depend on opening it. Its expected
SHA-256 is
`782d4fa8c8ab7e9aa38180c45f61532b66837af232422046ac5cb4ed18b20a44` so an
integrity check can detect accidental changes.

NovaFit v1 uses a new SQLite application database created solely from the new
project's Alembic migrations and authoritative source-controlled seed. It must use a
different path from the legacy database.
