# Lesson 5 notes: Implementing LLM Feedback Loops

My study notes for the solution notebook (code generation and debugging assistant). The original course notebook is
unchanged; my Claude version is `solution/lesson-5-implementing-llm-feedback-loops-claude.ipynb`.

## The task

The model writes a Python function `process_data(numbers, mode)`. Real tests, run by Python, decide whether the code
is correct, and the failures are sent back to the model so it can fix its own code.

## The intuition: generate, test, feed back, repeat

In lesson 4 a failed gate just dropped the claim. Here the failure becomes **information for the model**:

```
generate code → run tests → format failures → send back to the model → repeat (max 3 rounds, or until all pass)
```

Why this works:
- **The tests are an objective judge.** The model can't argue with a failing test, and nobody has to guess whether the code is right.
- **The feedback is specific.** "Test #6: inputs `([1, 3, 4], 'median')`, expected 3, actual None" tells the model exactly what to fix.
- **The model sees its own code plus the result.** It revises instead of starting from scratch.

Same principle as lesson 4: **the model reasons, code verifies.** The new part is that the verification result flows back into the model.

## The notebook, step by step

1. **Task and tests.** A requirements text plus 8 test cases (sum and average only).
2. **Initial generation.** The model writes the function, and the first version passes 8/8.
3. **Expanded tests (test-driven development).** The requirements grow, so the tests grow to 12:
   - a new `median` mode
   - non-numeric values are ignored
   - empty lists return `None`
   - an invalid mode raises `ValueError`

   The old code now fails 4 tests, because `median` isn't supported and a bad mode returns `None` instead of raising.
4. **One round of feedback.** Failures plus the current code go back to the model, which fixes all 4.
5. **The loop.** The same thing, up to 3 rounds from scratch, stopping early when 0 tests fail. Each round is stored in `iterations`, so you can compare them afterwards.

## The helper functions

| Function | Job |
|---|---|
| `get_completion` | Calls the LLM and returns the text |
| `extract_code` | Pulls the code out of the ```` ```python ```` fences in the reply |
| `execute_code` | Runs the generated code with `exec` and runs every test case |
| `format_feedback` | Turns the results into text for the model |

Details worth knowing:
- `execute_code` treats an expected exception as a pass (`expected: ValueError` passes if the function raises it).
- If the code can't even run, `format_feedback` sends the traceback and asks the model to fix the syntax or runtime errors.
- `exec` runs model-written code on my machine. That's fine for a course exercise, but in production it must be sandboxed.

## Claude version vs. original

Only the setup cells (2–4) differ; the prompts, tests and loop are identical.

| | Original | Claude copy |
|---|---|---|
| Client | `OpenAI(base_url=Vocareum)` | `anthropic.Anthropic(...)` with the workspace header |
| Model | `gpt-4.1-mini` | `claude-haiku-5-5` |
| System prompt | a `{"role": "system"}` message | separate `system=` argument (the helper converts it) |
| Keys | pasted into the notebook | loaded from `Udacity/.env` |

Unlike lesson 4, the helper does **not** strip the ```` ```json ```` fences here. `extract_code` needs the
```` ```python ```` fences, so the helper returns the text unchanged.

## Things to watch when running it

- `extract_code` is strict: it needs a line that is exactly ```` ```python ```` and a closing ```` ``` ````. If the model formats its reply
  differently, you get `ValueError: ... is not in list`. The fix is to make the extraction more forgiving.
- A stronger model may pass all 12 tests on the first revision, so the loop can end after one round. The saved
  original run needed three.
- Results vary between runs. The model is not deterministic, and that is why the loop has a maximum number of rounds.

## Takeaways
- Use tests as the objective judge, and write them first (test-driven development).
- Feed back specific failures (inputs, expected, actual), not just "it failed".
- Always cap the number of rounds, so the loop can't run forever or cost too much.
- Keep the history of every iteration, so you can see what changed and what stayed broken.
- Never run model-written code outside a sandbox in production.

## Link to the final project
The AgentsVille trip planner (`project/`) builds a multi-stage pipeline (chaining, lesson 4) and has a ReAct agent
that uses tools to improve the itinerary (lesson 2). I haven't worked through it yet. To be checked when I start it:
whether it also needs a check-and-revise loop like this lesson.
