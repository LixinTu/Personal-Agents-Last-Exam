# Environment Track proposal

**PR title format:** `[Environment Proposal] [Learning & research] Milepost practice platform (Khan Academy-inspired)`

Draft of 2026-11-19 (UTC), checked against the public runtime on `main` (release 1.5.0, commit `718a4cb`). Khan Academy is named only as a reference; no affiliation is claimed. All people, courses and results are synthetic.

## 1. App and domains

- **Authors / GitHub handles (required):** Lixin Tu (@LixinTu), proposed implementation and maintenance owner.
- **Reference app or website / URL (required):** Milepost, an independently branded synthetic practice platform; workflow reference: [Khan Academy](https://www.khanacademy.org)
- **Primary domain (required):** Learning & research
- **Secondary domains (optional):** None

### Domain names

One primary domain, spelled as in the [project domain list](../../README.md#environment-domains). The optional Calendar connection adds no secondary domain.

## 2. What you plan to build

**✍️ Your answer (required):** Milepost is a resettable practice platform that tracks one student's mastery skill by skill, so tasks can test whether an agent plans practice that fits this student and replans when results or dates change.

Study help differs from an errand: the same plan can be right for one student and useless for another, depending on what this student knows, what they have forgotten and when they can study. Milepost makes that checkable. For any plan it reports the level each covered skill is projected to reach by the date and, to the grader, every study rule the plan breaks. Tasks define success, including trade-offs and how much practice is too much. The projection uses a declared synthetic learner model and assumes planned practice is completed; it is not evidence of real learning gains.

Each instance holds one student account. The four workflows below are the acceptance baseline.

1. **Where the student stands.** Courses, units and skills with their prerequisites; the student's mastery of each skill and when it was last practised; assessments with their date, skills and a target level per skill; past results. Mastery fades with the time since practice, more slowly after spaced practice. An unknown ID returns `not_found`.
2. **Plan practice.** A study block has a date, a start, a length and one skill, and can be added, changed and removed. `plan.project` returns, for the stored plan and one assessment, the level each covered skill would reach by that date under the declared model version, assuming every planned block is completed in full. Broken study rules are not in its result; the plan checker reports them to the grader. Errors cover overlapping blocks, a block in the past and an unknown skill.
3. **Record practice.** A completed session arrives as an event with its skill and minutes and raises mastery, less so when the skill's prerequisites are weak. The student can also report practice done elsewhere; the agent records it with its source and time and can remove it but not edit it (a correction is remove, then record); removal undoes its effect but keeps later assessment results. Sessions that arrived as events cannot be removed. A report in the future or over a recorded session is refused. A planned block that passes without practice is marked missed.
4. **Replan after a change.** A posted result sets the mastery of the skills it measured. A moved or added assessment, or a change to the student's study rules, changes what can be reached. The plan is edited to fit, and `plan.project` shows what it still reaches. An event that names an unknown assessment or skill is refused and changes nothing.

The student's study rules (hours available, daily cap, rest days, block length) are readable. Milepost enforces only its own limits, such as no overlapping blocks. It does not stop a plan that breaks the student's rules or a record of practice that did not happen: every committed write goes to an evaluator-only activity log, and the task's grader decides.

Illustrative uses, not Task Track submissions:

- "Get me ready for Friday's quiz": plan only what this student needs. In the twin, the projection shows every covered skill above its target on Friday without practice, and the right answer is to say so and plan nothing.
- The exam moves three days earlier and two evenings are lost: plan what fits and say which skills will not be reached.
- "I did all four blocks": `practice.list` shows two. Check the times the student reports against the recorded sessions before adding any student-reported practice.

Each reference scenario has a twin: a copy of its seed in which one part of the student's record is changed, either mastery with the practice history that explains it, or a study rule. Each pair states which decision must change and why.

The mastery rule and scenario design come from my own pre-release [LearnerAgentBench](https://github.com/LixinTu/LearnerAgentBench), and the source version and changes will be recorded. The App is new code in the public App package format.

## 3. Potential app connections

- **Internal personal apps (required):** Calendar (optional)
  - No connection in the baseline. A task may have the agent check Calendar for conflicts and add study blocks with Calendar 1.3.0's own [tools](https://github.com/MatrAIx-ai/Personal-Agents-Last-Exam/blob/718a4cbc7709dee22bf1e4ad1cff984550d7e519/examples/apps/calendar/tools.json) (`calendar.freebusy`, `calendar.create`).
- **Other apps and websites (required):** None
  - No live Khan Academy API, accounts or real student data.

### Integration details

- **Operations and access (required):** 13 agent-facing MCP tools, prefixed `milepost.`: `courses.list`, `skills.list`, `mastery.get`, `assessments.list`, `practice.list|record|remove`, `plan.list|project|add_block|update_block|remove_block`, `preferences.get`. The `list`, `get` and `project` tools only read; the other five write. One reviewer renderer in `gui/` has four views (overview, skills and mastery, plan, practice log); reviewers are the only GUI audience the reference profile accepts. Infra operations are the runtime's `env.*` set (`env.load_seed`, `env.reset`, `env.apply_event`, `env.snapshot`, `env.event_status`, `env.validate_bundle`), which Core keeps off the agent connection.
- **State and lifecycle (required):** As in the public runtime, state is kept per run and App instance, and only Core commits, resets or snapshots it. Milepost's data has three parts: course data (the shared catalog), the student's record (the account's mastery, practice, plan and study rules), and evaluator-only records (the activity log and the student's learning parameters), which only `env.snapshot` returns (open question 1). A task loads its own seed, including practice history and past results, and its starting time (`virtual_now`) with `env.load_seed`; `env.reset` restores that seed and time for this App instance only, and `env.snapshot` returns the whole state. Each write takes a `request_id` and an `expected_version` and returns the committed result with the new version; a repeated `request_id` returns the first result, and a stale `expected_version` fails with `stale_version`. Mastery and missed blocks are one deterministic function of state and virtual time, computed when read; reads change nothing. Core moves the clock only when an event is applied and does not include it in tool results, so every Milepost read returns the current virtual time (open question 2). Times use the student's time zone.
- **Events and notifications (required):** Scenario events use the reference format and are applied with `env.apply_event`: `milepost.practice.completed` (a session is recorded), `milepost.assessment.result_posted` (sets the mastery of the measured skills), `milepost.assessment.changed` (an assessment is added or moved), `milepost.preferences.changed` (the student edits a study rule) and `milepost.time.advance` (only the clock moves). Task scenarios decide when they occur. Milepost neither publishes nor receives messages. Example input: `{"event_id": "ev-7", "at": "2027-03-02T21:00:00-05:00", "operation": "milepost.practice.completed", "payload": {"block_id": "blk-12", "skill_id": "alg-factoring", "minutes": 40}}`. Result: mastery of that skill rises, its last-practised time is set, and the block is marked done. The other four use the same envelope, with payloads `{assessment_id, results: [{skill_id, level}]}`, `{assessment_id, date, targets: [{skill_id, level}]}`, `{rule, value}` and `{}`.
- **Shared data and dependencies (required):** Stable IDs for the student and every object. `binding.json` maps the student to a person ID. Text only, with an empty asset catalog. Depends on Core's virtual time; no external services.

## 4. Construction plan and effort

**✍️ Work breakdown (required):**

| Workstream | Planned work and scale | Deliverable / quality check | Effort (person-days) |
| :--- | :--- | :--- | :--- |
| Web / app construction | App package under `apps/milepost/` in the public App package format: data model, 13 operations, 5 event handlers, the mastery rule, a plan checker (a pure function from a state snapshot to projected skill levels and broken study rules) and a reviewer renderer with 4 views. | Each workflow runs end to end through the tools, with its error cases. The renderer's output is tested against the tools' results. | 2–3 |
| Data and image collection | No crawling, images or lesson content. Skill lists are written by hand, following the tables of contents of openly licensed textbooks (e.g. OpenStax, CC BY 4.0), with attribution. | `contribution/dataset.json` lists every source and its reuse terms, with none unidentified. | 0.5–1 |
| Synthetic data and assets | Two fixtures that share three courses (algebra, statistics and biology; about 60 skills with prerequisites) and 6 assessments. Each fixture has its own fictional student, and the two differ in mastery, practice history and study rules. A seeded script builds each student's record from a short hand-written profile and derives mastery from the practice history with the mastery rule. Each reference scenario starts from one of the fixtures and has a twin seed. | `validateSeed` checks unique IDs, valid references, a mastery record for every skill, a skill graph without cycles and dates in order; a seed that fails is not loaded. I check every record by hand. No real people or schools. | 1–1.5 |
| App integrations | None in the baseline. `binding.json` and the `scenario` and `sandbox` policy profiles. | The agent connection lists only the tools its profile allows. | 0.5–1 |
| Validation and reset | (a) Business tests: every operation's success and error cases; the mastery rule at its boundaries.<br>(b) Core contracts, history loading and reset: a seed with practice history and past results loads at the task's `virtual_now`; a repeated `request_id`, a stale `expected_version` and a refused call leave business data unchanged; after `env.reset` the data hash and the clock equal those after `env.load_seed`.<br>(c) Four reference scenarios covering the workflows: the reference run and valid alternatives pass; scenario-specific no-action, over-planning and rule-breaking runs fail; a scripted planner that uses only what the agent can read is reported as a baseline.<br>(d) Twins: both members are validated and the decision that must change is checked; a claim that no plan fits both comes with a proof or an exhaustive check.<br>(e) Marker values planted in the evaluator-only records appear in no tool result, error or page.<br>(f) The public runtime's MCP verifier (`npm run app:verify -- --app apps/milepost`) and the repository's `public-runtime` and `base-runtime-compatibility` checks.<br>(g) Agent trials: 3 runs per reference scenario (12 in total) with a local open-weight model on a small harness I write, with traces and a short failure analysis; a usability check, not a model comparison.<br>(h) An independent reviewer reproduces one scenario from a clean checkout. | Test output and verifier reports covering permissions, state consistency and reset, with commands and versions; hashes, twin results, trial traces and reviewer notes in this PR. | 1.5–2.5 |

- **Milestones, owners and dependencies (required):** Owner: @LixinTu. Order: design review; the data model, mastery rule and workflows 1–2 with the first fixture; workflows 3–4 with the renderer, the second fixture and twins; then validation (a)–(h). Expected duration: a first runnable version within the 14 calendar days after plan approval that the contribution guide sets. Depends on the public App runtime on `main` (release 1.5.0), a platform dependency; if it changes, I update the thin layer that plugs Milepost into it. Overlap as of 2026-11-19 (UTC): the other open Learning & research environment proposals are #4 (Research Library, research records: no shared workflow), #51 (Language Study Workspace) and #67 (University Course Registration, enrolment: no shared workflow). #51 also has a study plan with a target date and replanning, for vocabulary cards; Milepost adds a skill graph with prerequisites, assessments with a target per skill, forgetting over time and twins. Related task proposals: #10 (a study-plan task), #34, #77, #29 and #52.
- **Resources and total effort (required):** 5.5–9 person-days, a rough estimate. Node.js 18+ on a laptop. Each seed stays under 1 MB, and the asset catalog is empty. Nothing paid is needed.
- **Scope boundary and open questions (required):** Not in scope: lesson content, tutoring conversations, answering practice items for the student, grading of written answers, goal tracking, teacher views, more than one student account per instance, real student data, and live sync with any product. The mastery rule is Milepost's own test rule; it is not Khan Academy's behaviour. Open questions for design review (if one is not answered at approval, the design above stands):
  - (1) May evaluator-only records stay in App data, returned only by `env.snapshot`? If not, which evaluator trace keeps the evidence graders need, given that Core's audit does not keep full call arguments?
  - (2) A task now sets the starting time with `virtual_now` in `env.load_seed`. Until the planned global scheduler exists, is an App event such as `milepost.time.advance` the right way to move the clock between sessions?
  - (3) Should study blocks reach Calendar through a platform-applied event, or only through the agent's own Calendar tools?
  - (4) Where should the plan checker live so that task authors can call it: in the App package or with the task files?
- **License and external dependencies (required):** Code and original synthetic fixtures under MIT, the repository's license. Adapted openly licensed text keeps its license and attribution, recorded in `dataset.json`. Material adapted from my own earlier benchmark is relicensed here under MIT. No Khan Academy content, logos or trademarks are redistributed. No external service is needed at run time.
