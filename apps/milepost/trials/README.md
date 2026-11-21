# Agent trials

A usability check of the environment with a small local model, as planned in the proposal (validation item g): 3 runs on the base member of each reference scenario. It shows where an agent gets stuck on Milepost's tools; it is not a model comparison, and one 7B model says little about stronger agents.

## Setup

- Model: `qwen2.5:7b-instruct` (7.6B, Q4_K_M) served by Ollama 0.40.14 on one laptop GPU (RTX 4070 Laptop, 8 GB), context 12,288 tokens, through its OpenAI-compatible endpoint with native tool calls. Temperature 0.6, seeds 1, 2 and 3, at most 900 tokens per reply, at most 25 model turns.
- Harness: [`run.cjs`](run.cjs). It starts Milepost through the public runtime, lets Infra load the scenario bundle and apply its setup, and gives the model the Agent connection's tools (dots in tool names become `__`) and the scenario's request. It fills `expected_version` and a fresh `request_id` for each write, so the model sees only business arguments; Core still checks both. Tool results longer than 6,000 characters are cut, and older results are shortened when the conversation passes 30,000 characters. A reply without a tool call ends the run. The grader is [`../examples/scenarios/grade.cjs`](../examples/scenarios/grade.cjs).
- Each run folder has `trace.jsonl` (every model reply, tool call and result) and `result.json` (grade, counts and the final message). Final snapshots are not kept: replaying a trace's tool calls on the scenario reproduces them.

```sh
~/.local/ollama/serve.sh &     # or any OpenAI-compatible server with tool calls
node apps/milepost/trials/run.cjs --scenario s1-cells-exam --seeds 1,2,3 --out apps/milepost/trials/round-1
```

## Round 1: environment at `3aef59e`

System prompt: "You are a study assistant for {student}, a student on the Milepost practice platform. You act for the student through the platform's tools. / The current time is {ask_at} (time zone America/New_York). Write times as ISO 8601 with the UTC offset, for example 2027-03-09T19:00:00-05:00. / Use the tools to read what you need and to make the changes the student asks for. When you are done, reply to the student in plain text without calling a tool."

**0 of 12 runs passed.**

| Scenario | Seed 1 | Seed 2 | Seed 3 |
| --- | --- | --- | --- |
| s1 cells exam | fail: targets not reached; 10 calls, 9 refused | fail: targets; 2 calls, 1 refused | fail: targets; 9 calls, 8 refused |
| s2 four blocks | fail: reports; 18 calls, 17 refused | fail: reports; 5 calls (4 wrong reports) | fail: reports; 1 call, nothing recorded |
| s3 exam moved | fail: rules, after exam, unreached practice; 3 calls | fail: same; 12 calls, 10 refused | fail: same; 39 calls, 35 refused |
| s4 check-in result | fail: targets; 24 calls, 18 refused | fail: rules, scope, minutes; 26 calls, 14 refused | fail: rules; 8 calls, 3 refused |

What went wrong:

1. **Invented IDs (9 of 12 runs, every s1, s3 and s4 run).** The model guessed IDs such as `cells_exam`, `cells_exam_monday_9`, `Biology` or `placeholder_skill_id` instead of listing assessments, courses or skills. Each guess returned `not_found` with "Assessment does not exist for this student.", which does not say where the valid IDs are. The model then retried guesses or told the student that the exam "has not been set up". Only 3 runs ever called `assessments.list`.
2. **Weekdays (all 3 s2 runs).** The request names days ("Tuesday's I did late"). Read results give ISO times only, and the model mapped the days to wrong dates (in the future, or Tuesday as 28 February). It recorded practice in the future (refused with "A reported session cannot end in the future.", 17 times in one run) or on the wrong days, and took skill IDs from old sessions instead of the week's blocks.
3. **Changing history and breaking rules (s4).** The model tried to edit done or missed blocks (`invalid_state`, `past_block`) and placed new blocks outside Daniel's study hours. This is a planning mistake rather than a gap in the tools: the refusals name the reason and `preferences.get` gives the hours.

Changes for round 2. In the environment: a `not_found` refusal names the read tool that lists the valid IDs; time refusals state the current time and weekday; blocks, sessions and assessments in read results carry their local weekday (`day`); tool descriptions say where each ID comes from. In the harness: the system prompt names the current weekday.
