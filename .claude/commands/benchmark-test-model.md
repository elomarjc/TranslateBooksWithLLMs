# Benchmark a model end-to-end: {{arg1}} / {{arg2}} [{{arg3}}]

Run the full TBL benchmark on a single model: produce translations on a
canonical pair set, manually evaluate per the rubric, and apply the scores.

**Args:**
- `{{arg1}}` = provider (`ollama`, `poe`, `openrouter`, or `openai`)
- `{{arg2}}` = model id (e.g. `gemma3:27b`, `gemini-3-flash-preview`, `mistral-medium-3.1`, `gpt-5-mini`)
- `{{arg3}}` = (optional) pair set: `quick` (8 pairs, default), `standard`
  (16 pairs), or `full` (28 pairs). See `benchmark/canonical_pairs.py` and
  `docs/BENCHMARK_WORKFLOW.md` for the lists and rationale.

If `{{arg1}}` or `{{arg2}}` is missing/invalid, ask the user via
AskUserQuestion before running. If `{{arg3}}` is missing, default to
`quick`. If `{{arg3}}` is provided but isn't one of `quick|standard|full`,
ask the user to pick.

Volume / time expectations:

- `quick`   → ~45 translations,  ~10 min judge time
- `standard` → ~125 translations, ~30 min judge time
- `full`    → ~245 translations, ~60 min judge time

---

## Step 1 — Load the binding references

Read these two files in full and keep them open in your working memory:

- [docs/JUDGE_RUBRIC.md](../../docs/JUDGE_RUBRIC.md) — penalty table, scale anchors, hard ceilings, dispersion rule.
- [docs/BENCHMARK_WORKFLOW.md](../../docs/BENCHMARK_WORKFLOW.md) — full procedure context.

Treat the rubric as binding; you must apply it strictly.

---

## Step 2 — Pre-flight: validate the model id exists

Cloud providers (especially Poe) reject unknown bot/model ids with a 404, and
the existing error path can fail with a misleading `'charmap' codec` message
on Windows — running the full benchmark just to discover this wastes ~7 minutes.

Validate the requested model **before** translating:

```
python -m benchmark.cli models -p {{arg1}} --check {{arg2}}
```

Three possible outcomes:

1. **Exit 0**: prints `OK: '<id>' is available on <provider>` → proceed to Step 3.
2. **Exit 1, "NOT FOUND"**: the id doesn't exist on the provider. The command
   prints up to 10 close matches. Surface the matches to the user via
   `AskUserQuestion` (offer the top 3 closest as options, mark the first as
   "Recommended"). Once the user chooses, treat that as the new
   `{{arg2}}` for the rest of the run. Don't guess silently.
3. **Exit 1, "could not fetch model list"**: missing API key or network issue.
   Stop and report the exact reason to the user (e.g. "POE_API_KEY is empty in
   .env"). Don't proceed.

For Ollama (local), this also catches "model not pulled" before wasting time.
For OpenRouter and OpenAI-compatible endpoints, the check uses each provider's
`/models` API.

---

## Step 3 — Produce translations

Resolve the pair set: if `{{arg3}}` is empty, use `quick`. Otherwise pass
`{{arg3}}` through to `--pair-set` (one of `quick`, `standard`, `full`).

Run via Bash:

```
python -m benchmark.cli run -p {{arg1}} -m {{arg2}} --no-evaluate --pair-set <quick|standard|full>
```

Wait for it to complete. Extract the `<RUN_ID>` from the line:

```
Results saved to: ...benchmark_results/<RUN_ID>.json
```

After completion, sanity-check the success rate via the run summary footer:

```
Success rate: 100.0%
```

**If success rate is < 90%**, something is wrong even though the run "completed".
Read a sample error from the JSON before evaluating:

```
python -c "import json; d=json.load(open('benchmark_results/<RUN_ID>.json', encoding='utf-8')); print(d['results'][0].get('error', 'no error'))"
```

Common causes and fixes:
- `'charmap' codec can't encode character '\\uXXXX'` → Windows console encoding
  bug. The CLI now reconfigures stdout/stderr to UTF-8 at startup, so this
  should not recur. If it does, confirm Python ≥ 3.7 and that no shim is
  reverting the encoding. As a last resort, prefix with `python -X utf8 -m ...`.
- HTTP 401 / 402 → API key invalid or insufficient credits. Stop and report.
- HTTP 404 (Poe) → bot id not found. This shouldn't happen if Step 2 passed;
  if it does, re-run Step 2 with the *exact* id from the error message.

**If the run fails to even start**, report the exact error to the user and
**stop**. Don't proceed to evaluation.

---

## Step 4 — Dump the evaluation brief

```
python scripts/dump_for_evaluation.py <RUN_ID> --batch-size 15 --out plan/eval_<RUN_ID>.md
```

The script writes one or more `plan/eval_<RUN_ID>_batch*.md` files. Read each
in full.

---

## Step 5 — Score each translation per the rubric

For every translation in every batch:

1. Identify all errors. Be explicit about them. Common categories:
   - Contresens / hallucinations (hardest hit: −2.0 accuracy)
   - Source word left untranslated in target (e.g. English in CJK output): −1.5 acc + −1.0 fluency
   - Mixed scripts in target (Traditional chars in zh-Hans output): −1.0 fluency
   - Grammar errors breaking the sentence: −1.5 fluency
   - Specialized terms wrong (botanical, scientific, marine, etc.): −0.5 to −1.0 acc
   - Lost rhetorical device (irony, parallelism, archaic register): −1.0 to −1.5 style
   - Calques/anglicisms with native equivalents: −0.5 fluency

2. Compute scores:
   - `accuracy`, `fluency`, `style` start at 10, deduct the penalties.
   - `overall` is a holistic call constrained by **rubric §4**: cap at 9.0
     without a published human reference; if any dimension < 6.0, overall ≤ 6.0;
     overall must not exceed `min(accuracy, fluency, style) + 0.5`.

3. Don't be charitable. Most LLM scores fall in 6.5–8.5 on this rubric. Reserve
   9.0 for genuinely outstanding work approaching a published human translation.

4. Write a 1–2 sentence `feedback` documenting the deductions with explicit
   penalty values (e.g. "−2.0 acc, −1.0 style"). The feedback is auditable and
   ends up in the wiki.

---

## Step 6 — Write the JSON reply

Single file: `plan/eval_<RUN_ID>_rubric_v1.json`

Format:
```json
[
  {"eval_id": "<10-hex>", "scores": {
    "accuracy": 0.0,
    "fluency": 0.0,
    "style": 0.0,
    "overall": 0.0,
    "feedback": "..."
  }},
  ...
]
```

One object per `eval_id` from the brief. **Don't omit any.** Use the Write tool.

---

## Step 7 — Apply the scores

Determine the judge model:
- If the current Claude Code session is Opus 4.7 → `claude-opus-4-7-rubric-v1`
- If Sonnet 4.6 → `claude-sonnet-4-6-rubric-v1`
- If Haiku 4.5 → `claude-haiku-4-5-rubric-v1`

(See the `claude-opus-4-7` etc. model IDs in the system context.)

Run:
```
python scripts/apply_evaluations.py <RUN_ID> plan/eval_<RUN_ID>_rubric_v1.json --judge-id <judge-id>
```

The script is idempotent — safe to re-run.

---

## Step 8 — Report results

Generate a final report for the user with:

1. **Header**: `<RUN_ID>`, model `{{arg2}}` via `{{arg1}}`, `<N>` translations evaluated.
2. **Per-pair averages**: compute via inline Python from `benchmark_results/<RUN_ID>.json`. Format as a markdown table:
   | Pair | N | Avg overall |
3. **Global average overall** + range (best, worst).
4. **Top 3 best** translations (with eval_id, text_id, target_lang).
5. **Top 3 worst** translations (with eval_id, text_id, target_lang).
6. **Notable failure modes** detected across the run (e.g. "consistent botanical errors on Wilde", "hallucinated place name in KO→EN").
7. **Next-step suggestion** with copy-pasteable command:
   ```
   python -m benchmark.cli submit benchmark_results/<RUN_ID>.json --by github:hydropix --provider {{arg1}} --judge-id <judge-id>
   ```

---

## When evaluating multiple models in one session (rubric §5)

If the user has previously run this skill in the same session for other models
on the same 8 pairs, also apply **comparative dispersion**:

1. Group results by `(text_id, target_lang)` triples.
2. Across the N model outputs for each triple, rank them 1st through Nth.
3. Enforce **≥0.3 points difference** between adjacent ranks on `overall`.
4. Document ties explicitly in feedback ("tied with model X on this triple").

This forces visible spread in the wiki tables and prevents flattening.

---

## Important guardrails

- **The 8 canonical pairs are fixed.** Don't substitute. Comparability across
  models depends on this set staying the same.
- **The judge_id format is `<judge-model>-rubric-v<n>`.** Don't deviate.
- **Use TodoWrite** to track the 8 steps as you proceed (validate → translate
  → dump → score → write JSON → apply → report).
- **Don't auto-submit.** The skill stops at the report. The user decides
  whether to commit and submit separately.
