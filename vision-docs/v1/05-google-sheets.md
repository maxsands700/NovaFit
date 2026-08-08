# Google Sheets Integration

Google Sheets is NovaFit's workout-execution surface. It should make a
prescribed workout easy to follow and quick to log on a phone or computer. The
athlete should not need to interpret progression rules, edit a prescription, or
fill out a general check-in.

NovaFit remains the source of truth. The application owns the program,
prescriptions, completed-workout history, progression decisions, and every
machine identity used to connect a Sheet row to an exercise. Sheets displays a
published workout and collects the athlete's execution evidence.

## Workbook

Each athlete receives one NovaFit-managed workbook. v1 needs one visible
`Workout` tab and one hidden `_NovaFit` tab.

- `Workout` is the athlete-facing current workout.
- `_NovaFit` stores the schema version, athlete and publication identities,
  exercise and set identities, prescribed values, row hashes, submission state,
  and import receipts. It is never an athlete input surface.

The workbook contains only the currently actionable workout. NovaFit publishes
the next workout after it has processed the completed one. It does not expose a
future program calendar that could become stale after a progression decision.

## Workout Layout

The visible grid follows the supplied Excel template. One exercise is a block
of one row per prescribed set. The first row contains the labels; every
exercise block uses the same columns:

| Exercise | Tempo | Prescription | Outcome | Completed Reps | RPE | RIR | Skip Reason | Pain | Technique Notes |
|---|---|---|---|---:|---:|---:|---|:---:|---|
| Back Squat | 10X0 | 5 reps @ 225 lb |  |  |  |  |  | ☐ | Proud chest, braced core, explode from the hole |
|  |  | 5 reps @ 225 lb |  |  |  |  |  | ☐ |  |
|  |  | 5 reps @ 225 lb |  |  |  |  |  | ☐ |  |

`Exercise`, `Tempo`, and `Technique Notes` are each merged vertically across
the exercise's complete set block and centered both horizontally and
vertically. `Prescription` remains one read-only cell per set so the prescribed
reps and load stay adjacent to the athlete's result. A future exercise with a different
number of sets receives a block of that many rows; it is not forced into a
three-row shape.

Exercise, tempo, prescription, and technique notes are NovaFit-owned and
read-only. The athlete fills out only:

1. `Outcome` — `COMPLETED` or `SKIPPED`
2. `Completed Reps`, `RPE`, and `RIR` when the outcome is `COMPLETED`
3. `Skip Reason` when the outcome is `SKIPPED`: `PERFORMANCE_LIMIT`, `PAIN`, or
   `NON_PERFORMANCE`
4. `Pain` — checkbox when pain occurred during an otherwise completed set

Every submitted row must have a terminal outcome. For `COMPLETED`, completed reps,
RPE, and RIR are required; `Skip Reason` must be blank. For `SKIPPED`, a skip reason
is required and completed-rep/RPE/RIR/Pain fields must be blank. A skipped exercise
is a block whose every row is skipped; an abandoned workout is a valid partial submission
only after its remaining rows are explicitly skipped. An unsubmitted blank workout
or exercise is `NO_OBSERVATION`, not failure.

The sheet displays the expected RPE/RIR counterpart while either is entered;
`abs(RPE - (10 - RIR)) > 0.5` requires correction. The athlete always performs the
published prescription at its stated load; actual load is not an athlete-input field.
The importer derives `INCOMPLETE_SET` when `completed_reps < prescribed_reps`.
`completed_reps > prescribed_reps` is an undeclared prescription and requires a
correction. `PERFORMANCE_LIMIT` and `INCOMPLETE_SET` are performance evidence;
`NON_PERFORMANCE` is non-comparable. `PAIN`, whether selected as a skip reason or
checked on a completed set, initiates the pain-hold rule below. Normal exertion and
ordinary muscle soreness are not pain.

Technique notes are read-only catalog guidance. NovaFit neither asks the athlete to
report technique breakdown nor evaluates form in v1; every submitted completed rep
is accepted under the athlete's good-form disclaimer.

## Formatting

The Workbook should match the supplied Excel template's compact, bordered
exercise-block layout:

- use Arial at the template's normal size;
- use thin cell borders around each exercise block and its header row;
- center the compact numeric and checkbox columns; wrap long technique notes;
- retain the template's column proportions, including a wide Exercise column
  and a wider Technique Notes column; and
- keep every cell in an exercise block visually continuous, including the
  merged cells.

NovaFit replaces the template's uncoloured presentation with its theme only:
black for the primary header and strong boundaries, gold for prominent actions
and active emphasis, grey for muted or read-only prescription cells, and white
for the normal workout surface. Gold and black must retain legible contrast;
colour is never the only way to communicate an editable cell or a pain signal.

Editable cells should be visually distinct from NovaFit-owned cells while
remaining quiet enough that the prescription is easy to scan. `Outcome` and `Skip
Reason` are validated dropdowns; `Pain` is a checkbox. No free-text explanation is
required for v1.
The `Submit Workout Completion` action follows the same NovaFit black-and-gold
treatment and remains below the final exercise block.

## Publication and Import

```
publish_workout(athlete, scheduled_session):
  require a current approved NovaFit prescription
  render its exercise blocks into Workout
  store immutable publication and row identities in _NovaFit
  mark the workbook DRAFT

submit_workout(workbook):
  verify workbook schema, athlete, publication identity, and row hashes
  read Outcome, Completed Reps, RPE, RIR, Skip Reason, and Pain from each set row
  reject changed NovaFit-owned cells and formulas in athlete-input cells
  require every submitted row to have a valid terminal outcome
  require fields appropriate to the selected outcome
  derive INCOMPLETE_SET when completed_reps < prescribed_reps
  require correction when completed_reps > prescribed_reps
  create an immutable completed-workout revision
  import it once, even if the submission is retried
  mark the workbook IMPORTED
```

The importer never derives exercise identity, target reps, tempo, or technique
from display text. It uses the protected `_NovaFit` identities and verifies
that the visible worksheet still corresponds to the published prescription.
Inserted or deleted rows, changed headers, altered merged layout, or modified
read-only prescription cells fail closed and require republishing rather than
guessing what the athlete meant.

A submission records a complete snapshot of the visible athlete inputs. A
later correction creates a new completed-workout revision; it never overwrites
the imported history. Repeating the exact same submission is idempotent.

## Google Access and Credentials

NovaFit uses the Google Sheets API through a dedicated Google Cloud service
account. The service
account is the workbook publisher and importer; athletes do not need to grant
NovaFit access to their personal Google account.

Setup is explicit:

1. Create a dedicated NovaFit Google Cloud project and enable the Google Sheets
   API.
2. Create a dedicated service account for NovaFit's Sheets integration and
   create its JSON key.
3. Create or choose the athlete's workbook, then share that single spreadsheet
   with the service account's email address using the minimum role that permits
   NovaFit to publish, format, and import it.
4. Configure NovaFit with that spreadsheet ID and the athlete it belongs to.

The service-account key stays outside the repository, is readable only by the
NovaFit application user, and is supplied through deployment configuration or a
local protected path. It is never committed, placed in a workbook, or exposed
to the athlete. NovaFit requests only the Google Sheets API scope; it does not
use broad Google Drive access. Application Default Credentials are disabled
unless deliberately enabled for a controlled development environment.

Workbook sharing is not NovaFit authentication. NovaFit still verifies the
workbook's schema, athlete binding, publication identity, and protected
machine-only records before it accepts a submission.

## Progression Handoff

After a successful import, NovaFit evaluates each exercise independently. A fully
completed exercise can update evidence even when another exercise is skipped. An
`INCOMPLETE_SET` or `PERFORMANCE_LIMIT` is failure evidence; `NON_PERFORMANCE` is
non-comparable. Pain takes precedence: NovaFit puts the program in `PAIN_HOLD`,
publishes no next workout, and awaits either an auditable correction or the athlete's
confirmation that the report stands. Confirmation ends the program as
`PAIN_REPORTED`; NovaFit does not provide injury programming or substitutions.
Missing or inconsistent RPE/RIR returns `NEEDS_CORRECTION` and blocks the next
prescription; the original submission and its correcting revision remain auditable.

The athlete's workflow stays deliberately small:

1. Open the published workout.
2. Perform each prescribed set.
3. Mark each row completed or skipped; enter the required result or skip reason.
4. Check Pain when it occurred during a completed set.
5. Submit the completed workout and await the next published session.
