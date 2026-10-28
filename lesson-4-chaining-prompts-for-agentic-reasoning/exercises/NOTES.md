# Lesson 4 notes: Chaining Prompts for Agentic Reasoning

My study notes for the solution notebook (claim triage). The original course notebook is unchanged; my Claude version is
`solution/lesson-4-chaining-prompts-for-agentic-reasoning-claude.ipynb`.

## The task

Turn free-text car insurance claims (First Notice of Loss, FNOL) into a routing decision: which work queue
(`glass`, `fast_track`, `material_damage`, `total_loss`) and which priority (1–5).

## The intuition: a chain instead of one big prompt

A single prompt that extracts, judges, estimates and routes all at once has three problems:

- **Hard to debug.** When the final queue is wrong, you can't tell which step failed.
- **Mistakes spread silently.** An early misreading distorts every later decision, and nothing catches it.
- **Divided attention.** One prompt juggling five jobs does each of them worse.

The fix is to split the task into small steps with one job each, and put a code check (a "gate") after every step.

```
FNOL text → [1. Extract] → Gate 1 → [2. Assess severity] → Gate 2 → [3. Route] → Gate 3 → results table
```

This is the same pipeline a real insurance office runs: intake clerk, then damage assessor, then dispatcher.
Each prompt plays that role, and each step gets only what it needs, already structured.

**Core principle: the model reasons, code enforces.** The LLM does the fuzzy work (reading human language,
judging how bad the damage is). Plain Python does the strict work (validating, checking ranges, deciding
whether to continue).

## The stages and their gates

Steps pass JSON to each other, not prose. Each Pydantic model is a contract for one step's output, and the
gate after the step enforces it. Prose can't be checked automatically; `{"severity": "Moderate", "est_cost": 2500}` can.

### Stage I: Extraction → Gate 1 (format)
- **Prompt:** extract `claim_id`, `name`, `vehicle`, `loss_desc`, `damage_area` as JSON.
- **Gate 1 (`gate1_validate_claim_info`):** checks valid JSON, field lengths, and that `damage_area` only uses
  allowed values. It catches made-up categories like `"taillight"`.

### Stage II: Severity → Gate 2 (logic)
- **Prompt:** apply the insurer's rules and return `severity` + `est_cost`.
- **Gate 2 (`gate2_cost_range_ok`)** runs three checks:
  1. Is it valid JSON?
  2. Does it have the right shape? `severity` must be exactly `Minor`/`Moderate`/`Major`, and `est_cost` must be > 0.
  3. **Is it consistent?** The cost must fit the severity:

     | Severity | Allowed cost |
     |---|---|
     | Minor | $100 – $1,000 |
     | Moderate | $1,000 – $5,000 |
     | Major | $5,000 – $50,000 |

  Check 3 is the important one. `{"severity": "Minor", "est_cost": 3000}` is perfectly formatted but
  contradicts itself, and only a rule in code catches that.

### Stage III: Routing → Gate 3 (action)
- **Prompt:** given the claim and its severity, pick a queue and a priority.
- **Gate 3 (`gate3_validate_routing`):** the queue must be one of the 4 allowed values and the priority must be 1–5,
  because a real downstream queue system would break on anything else.

### When a gate fails
The error is printed with the model's raw response and the function returns `None`. Later stages skip `None`,
so bad data stops at the gate and never moves forward quietly. The failure message tells you which step to fix.

## Claude version vs. original

Cells 2–4 (setup) and cell 6 (sample claims, see below) differ; prompts, gates and stages are identical.

| | Original | Claude copy |
|---|---|---|
| Client | `OpenAI(base_url=Vocareum)` | `anthropic.Anthropic(...)` |
| Model | `gpt-4.1-mini` | `claude-haiku-5-5` (small, cheap) |
| System prompt | a `{"role": "system"}` message | separate `system=` argument (the helper converts it) |
| JSON | `response_format={"type": "json_object"}` | no JSON mode; the prompts say "only JSON" and the helper strips ```json fences |
| Keys | pasted into the notebook | loaded from `Udacity/.env` |

**Key setup gotcha:** my Console creates user-level keys (`sk-ant-usr-...`). These must send the workspace ID on
every request, or the API returns a 400. The client cell does this with
`default_headers={"anthropic-workspace-id": os.environ["ANTHROPIC_WORKSPACE_ID"]}`, and the ID is stored in `.env`.
After editing `.env`, restart the kernel.

## My own sample claims (Claude copy)

I replaced the course's 5 FNOLs with new cases. Same structure, and they still cover every queue:

| Claim | Case | Expected route | What it tests |
|---|---|---|---|
| C001 Priya Patel, 2017 Mazda CX-5 | Gravel chips the windshield | `glass` (Minor) | glass-only rule |
| C002 Lucas Moreau, 2021 Hyundai Elantra | Low-speed rear-end, bumper + grille, drivable | `material_damage` (Moderate) | "grille" isn't an allowed `damage_area`, so does the model map it to `front`/`bumper` or does Gate 1 catch it? |
| C003 Aisha Okafor, 2023 Chevrolet Silverado | Deer at highway speed, engine dead, towed | `total_loss` (Major, priority 1–2) | high-severity path |
| C004 Tomasz Kowalski, 2016 VW Golf | Storm branch: roof, trunk, quarter panel, rear window | `material_damage` or `total_loss` | borderline case: multiple panels |
| C005 Sofia Lindqvist, 2022 Kia Sportage | Shopping cart: shallow dent + scratches on one door | `fast_track` (Minor) | the queue the course samples never hit |

Results can vary between runs and models, especially for borderline cases like C004. That's expected, and it's
exactly why the gates check consistency rather than specific answers.

## Takeaways
- Break complex tasks into single-purpose prompts.
- Pass structured data (JSON + schemas) between steps so you can validate it.
- Gate every LLM step with deterministic code, and check consistency, not just format.
- Fail loudly at the step that broke.

## Link to lesson 5
Here a failed gate just drops the claim. Lesson 5 (feedback loops) sends the gate's error back to the model,
for example "Minor must cost $100–1,000, you said $3,000", so the model can fix its own answer. That turns a gate
into a self-correcting loop, a core building block of agents.
