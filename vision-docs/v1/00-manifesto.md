# NovaFit: Overarching Goal
Training is simple in theory. All the athlete has to do train for a specific adaptation by following these rules:

1. Pick clear goals.
2. Choose exercises that directly serve those goals.
3. Apply progressive overload.
4. Recover enough to adapt.
5. Track results.
6. Adjust only when the current plan stops working.

Items 1 and 4 are the athlete's responsibility. In v1, NovaFit does not monitor or manage sleep or recovery; it assumes the athlete is recovering sufficiently. NovaFit exists to make the remaining work more reliable and less time-consuming for athletes that do not have a coach:

2. **Choose appropriate exercises.** Exercise selection is straightforward for some goals—for example, bench pressing to improve a 315 lb bench. It is more nuanced in calisthenics. An athlete pursuing a freestanding handstand push-up may choose a movement that is too difficult and repeatedly fail maximal attempts, or one that is too easy to provide an effective strength stimulus. NovaFit should select an appropriate prerequisite exercise: one the athlete can perform cleanly while progressively overloading it toward the goal.

3. **Apply progressive overload.** Athletes can underdose or overdose training based on bias or subjective feel. Strength-training rules are sufficiently structured to encode in deterministic software.

5. **Track results.** Recording workouts, load, volume, and related evidence is tedious manually but simple for a database.

6. **Adjust when needed.** Software can use recorded evidence to identify when a plan is no longer working more consistently than an athlete relying solely on subjective feel.

The purpose of NovaFit is to perform items 2, 3, 5, and 6 and communicate them to the athlete in an easy way. Athletes do not want to waste their time. NovaFit should make it extremely simple for the athlete. All the athlete should have to do is follow the prescribed workout given to them, log the results, and await for the next scheduled workout.

# The "Source of Truth" / Data
Version 1.0 of NovaFit will focus on **general strength training** with the exact
barbell, dumbbell, pull-up, dip, and bodyweight-core exercises defined in
`08-exercise-catalog.md`. Pull-ups and dips may use external load without becoming
separate exercise identities. Later, we can add:
1. Weighted Hypertrophy Training
2. Calisthenics Strength Training
3. Calisthenics Hypertrophy Training
4. Olympic Lifting Power Training
5. Rock Climbing Strength Training
6. Running Sprint & Endurance Training
7. Swimming Sprint & Endurance Training
8. Hybrid Training
9. ...

For general strength and hypertrophy programming, we will treat Overcoming Gravity, 2nd Edition by Steven Low as the bible. While it is intended for Calisthenics, it contains the broad strength and hypertrophy programming principles that can be applied to weighted strength and hypertrophy training. The EPUB of the book, and my concise, actionable synthesis for general programming are found in `data/OG2/`

For the future training types, we will use other sources of data. I will add these at a later point.

## Future Product Scope

Version 1 deliberately makes prescriptions from athlete goals, capabilities, and
logged training performance. Later versions should be able to add more context and
reduce the manual work surrounding training through optional integrations:

1. **Google Calendar** — publish scheduled workouts and keep calendar events aligned
   when NovaFit reschedules or revises a session.
2. **WHOOP and other recovery sources** — ingest sleep, recovery, and readiness
   evidence so a future, explicitly versioned policy can evaluate how recovery
   relates to training performance and progress.
3. **Nutrition tracking** — ingest nutrition evidence and evaluate how it relates to
   recovery and progress without making a particular nutrition vendor part of the
   coaching model.
4. **Additional training adaptations and domains** — add the hypertrophy,
   calisthenics, Olympic lifting, climbing, endurance, swimming, and hybrid training
   types listed above without rewriting weighted-strength behavior.

These are roadmap capabilities, not hidden v1 inputs. Until a policy explicitly
supports one of these signals, NovaFit may store and display it but must not silently
use it to alter a prescription. External platforms are data sources and delivery
channels; NovaFit remains the source of truth for programs, decisions, and workout
history.
