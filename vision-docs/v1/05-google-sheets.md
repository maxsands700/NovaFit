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

| Exercise | Tempo | Target Reps | Completed Reps | RPE | RIR | Failed Reps | Pain | Technique Notes |
|---|---|---:|---:|---:|---:|:---:|:---:|---|
| Back Squat | 10X0 | 5 @ 225 lb |  |  |  | ☐ | ☐ | Proud chest, braced core, explode from the hole |
|  |  | 5 @ 225 lb |  |  |  | ☐ | ☐ |  |
|  |  | 5 @ 225 lb |  |  |  | ☐ | ☐ |  |

`Exercise`, `Tempo`, and `Technique Notes` are each merged vertically across
the exercise's complete set block and centered both horizontally and
vertically. `Target Reps` remains one read-only cell per set so the prescribed
dose stays adjacent to the athlete's result. A future exercise with a different
number of sets receives a block of that many rows; it is not forced into a
three-row shape.

Exercise, tempo, target reps, and technique notes are NovaFit-owned and
read-only. The athlete fills out only:

1. `Completed Reps`
2. `RPE`
3. `RIR`
4. `Failed Reps` — checkbox
5. `Pain` — checkbox

The athlete may leave RPE and RIR blank when they genuinely do not know them.
NovaFit stores that as unknown rather than inventing an effort value. A checked
`Failed Reps` box records that the set included one or more failed reps. A
checked `Pain` box is a safety signal: NovaFit must hold automatic progression
for that exercise and direct the athlete to use safe judgment before continuing.
Normal exertion and ordinary muscle soreness are not pain.

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
colour is never the only way to communicate an editable cell, a failed rep, or
a pain signal.

Editable cells should be visually distinct from NovaFit-owned cells while
remaining quiet enough that the prescription is easy to scan. Checkboxes are
used for `Failed Reps` and `Pain`; no text such as `Yes` or `No` is required.
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
  read only the five athlete-input fields from each set row
  reject changed NovaFit-owned cells and formulas in athlete-input cells
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

After a successful import, NovaFit evaluates every exercise independently using
the completed reps, effort evidence, failed-rep signal, and pain signal. The
rules in `04-progression-policies.md` determine whether the next exposure
progresses, stays the same, regresses, or stops. A checked pain box takes
precedence over normal progression. Missing RPE or RIR can limit confidence in
a progression decision, but should not cause the system to fabricate data.

The athlete's workflow stays deliberately small:

1. Open the published workout.
2. Perform each prescribed set.
3. Enter completed reps and, when known, RPE and RIR.
4. Check Failed Reps or Pain only when they occurred.
5. Submit the completed workout and await the next published session.
