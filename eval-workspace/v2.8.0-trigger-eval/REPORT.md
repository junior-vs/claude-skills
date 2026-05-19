# v2.8.0 Description-Triggering Eval — Anthropic skill-creator methodology

**Date:** 2026-05-19
**Tool:** Anthropic's `skill-creator` (cloned from `github.com/anthropics/skills`)
**Methodology:** Phase 4 — description triggering accuracy

Tests whether each v2.8.0 SKILL.md's `description` field correctly triggers Claude to consult the skill on relevant queries, and correctly does NOT trigger on adjacent/unrelated queries.

## Setup

- For each of 3 representative skills (process-mapper, deal-desk, commercial-forecaster), wrote 10 trigger queries: 5 should-trigger + 5 should-not-trigger
- Should-not-trigger queries are deliberate near-misses (adjacent commercial / engineering / finance work that shares keywords)
- Used `python3 -m scripts.run_eval` from Anthropic's skill-creator with `--runs-per-query 1`
- The eval runs `claude -p` against each query, detects whether the skill triggers via stream-json events

## Aggregate results

| Skill | Pass rate | False positives | False negatives |
|---|---|---|---|
| `process-mapper` | 7/10 (70%) | 0 | 3 |
| `deal-desk` | 7/10 (70%) | 0 | 3 |
| `commercial-forecaster` | 8/10 (80%) | 0 | 2 |
| **AGGREGATE** | **22/30 (73%)** | **0** | **8** |

## Key insights

### ✅ Zero false positives across all 3 skills

Every should-not-trigger query correctly did NOT trigger the skill — even on adjacent queries that share keywords (e.g., commercial-forecaster on "deal-desk discount approval", deal-desk on "Van Westendorp pricing", process-mapper on "Erlang-C capacity planning"). **The `Distinct from` sections are working** — boundaries are clean.

### ⚠️ 8 false negatives — descriptions undertrigger on legitimate queries

The descriptions are too narrow / not "pushy" enough. Per the skill-creator playbook:

> Make descriptions **"pushy"** to combat undertriggering. Include both the action AND specific triggering contexts.

#### Failures per skill

**`process-mapper` (missed 3):**
- "I want to understand our incident-response cycle time — P50 and P90 per stage, plus value-add ratio" — *uses "cycle time" + "P50/P90" + "value-add ratio" — exact tool output, but description doesn't say "compute" or "measure"*
- "Our customer-onboarding takes too long. Walk me through measuring the bottleneck and computing cycle time per stage."
- "BPMN diagram for our quote-to-cash process with swim lanes" — *short query without "process improvement" framing*

**`deal-desk` (missed 3):**
- "Review this MSA before we sign — there's MFN pricing, auto-renew without notification, and the DPA section is missing. Customer is EU-based."
- "What's the redline severity on uncapped indemnity? Need the standard counter-language and who has to sign."
- "Help me build a discount approval workflow for our deal desk — who routes what based on size + discount %?"

**`commercial-forecaster` (missed 2):**
- "We have a $4.2M pipeline. Forecast bookings for next quarter using last 4 quarters of conversion data, weighted toward recent."
- "Detect leaky cohorts before they hit the consolidated NRR number — give me the cohort-level decomposition"

## Pattern analysis

The failure mode is **specificity overshooting**: each description is heavy on persona ("when a Head of People Ops, BizOps lead, or Internal Communications owner needs to..."), but light on **direct trigger phrases the user would actually say** (e.g., "BPMN diagram", "review my MSA", "leaky cohort").

The user spec ("avoid generic skills") pushed us toward persona-heavy descriptions. This trades 0 false-positives (excellent) for ~25% false-negatives (suboptimal).

## Recommendation for Sprint 4 (optional)

Run the description-optimization loop on the 3 skills:

```bash
cd eval-workspace/anthropic-skill-creator/anthropic-skills/skills/skill-creator
python3 -m scripts.run_loop \
  --eval-set eval-workspace/v2.8.0-trigger-eval/eval-sets/process-mapper.json \
  --skill-path business-operations/skills/process-mapper \
  --max-iterations 5
```

This will:
1. Split eval set 60% train / 40% test
2. Run baseline against current description
3. Call Claude to propose improved descriptions based on failure patterns
4. Re-evaluate iteratively, picking best description by test-set score (avoids overfitting)

Expected outcome: pass rate 73% → 90%+ after 5 iterations, with 0 false positives preserved.

## Files

- `eval-sets/process-mapper.json` — eval queries
- `eval-sets/deal-desk.json`
- `eval-sets/commercial-forecaster.json`
- `process-mapper-eval.json` — full results
- `deal-desk-eval.json`
- `commercial-forecaster-eval.json`

## Skills NOT evaluated yet (12 of 15 v2.8.0)

To complete the v2.8.0 trigger-accuracy audit, run the same eval against:

**business-operations/** — capacity-planner, internal-comms, knowledge-ops, procurement-optimizer, vendor-management, business-operations-skills (orchestrator)

**commercial/** — pricing-strategist, partnerships-architect, channel-economics, commercial-policy, rfp-responder, commercial-skills (orchestrator)

Estimated time: ~3 minutes per skill × 12 skills = ~36 minutes total.
