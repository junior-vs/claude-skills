# How to reproduce / extend this eval

The eval uses Anthropic's official `skill-creator` skill (Phase 4: description-triggering accuracy).

## Setup (once per machine)

```bash
# Clone Anthropic's skill-creator (gitignored at this path)
mkdir -p eval-workspace/anthropic-skill-creator
cd eval-workspace/anthropic-skill-creator
git clone --depth 1 https://github.com/anthropics/skills.git anthropic-skills
```

## Run a single skill eval

```bash
cd eval-workspace/anthropic-skill-creator/anthropic-skills/skills/skill-creator

python3 -m scripts.run_eval \
  --eval-set /home/user/claude-skills/eval-workspace/v2.8.0-trigger-eval/eval-sets/process-mapper.json \
  --skill-path /home/user/claude-skills/business-operations/skills/process-mapper \
  --runs-per-query 1 \
  --num-workers 2 \
  --timeout 60 \
  > /tmp/process-mapper-eval.json
```

## Output schema

The `run_eval.py` script writes JSON to stdout with this structure:

```json
{
  "skill_name": "process-mapper",
  "skill_path": "...",
  "description": "...",
  "trigger_threshold": 0.5,
  "results": [
    {
      "query": "...",
      "should_trigger": true,
      "trigger_rate": 1.0,
      "triggers": 1,
      "runs": 1,
      "pass": true
    }
  ],
  "summary": {
    "total": 10,
    "passed": 7,
    "failed": 3
  }
}
```

## Run the optimization loop (Phase 4 — automated description rewriter)

```bash
python3 -m scripts.run_loop \
  --eval-set eval-sets/process-mapper.json \
  --skill-path /home/user/claude-skills/business-operations/skills/process-mapper \
  --model claude-opus-4-7 \
  --max-iterations 5 \
  --verbose
```

This will:
1. Split eval set 60% train / 40% test
2. Evaluate current description (3 runs/query for reliability)
3. Call Claude to propose improved descriptions based on failure patterns
4. Iterate up to 5 times
5. Output `best_description` selected by test score (avoids overfitting)

## Authoring new eval sets

Each eval set is a JSON array of 16-20 queries (8-10 should-trigger + 8-10 should-not-trigger). Per Anthropic's skill-creator playbook:

- **Should-trigger** — different phrasings of the same intent, formal + casual variants, cases where skill isn't explicitly named but clearly needed
- **Should-not-trigger** — near-misses (share keywords but need something different), adjacent domains, ambiguous phrasing
- **Avoid** — trivially-easy negatives like "write fibonacci" for a PDF skill
