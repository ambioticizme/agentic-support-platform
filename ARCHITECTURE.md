# Architecture Notes

This document captures the architectural decisions I made building the platform that I'd want to explain in an interview — the ones that came from operating the system, not designing it on a whiteboard. Most AI support systems fail because they're built by people who haven't done support. The decisions below are what twenty years of support engineering instinct looks like applied to multi-agent design.

---

## 1. Hard business rules run before the LLM ever sees a ticket

**The decision.** Refund eligibility, tier-based routing, churn-risk escalation, named-account handling — none of these are LLM calls. They're a deterministic rule layer that executes first, with confidence 1.0, and only the leftover ambiguity gets handed to Claude.

**Why it matters.** The default pattern in AI tutorials is "let the agent figure it out": give the LLM the ticket plus the policy and let it reason its way to an answer. That feels AI-native. It's also how you get a model confidently telling a customer their refund is approved when it isn't.

The rules layer guarantees that compliance-sensitive logic is reproducible. Same inputs, same output, every time. When a customer escalates and asks why they were denied, the answer is a function call you can re-run. There's no "the model said so" stage.

**Worked example — refund eligibility.** A two-stage agent:

- **Stage 1 (LLM).** A binary intent classifier reads subject + body and returns `is_refund / confidence / reasoning`. Some customers pick "refund" on the form when they're really asking about billing or trial extensions. Stage 1 catches misfiles and routes them to Customer Success without ever reaching the eligibility tree.
- **Stage 2 (pure function).** Maps `(account_tier, subscription_start_date, reseller_domains)` to one of six outcomes: eligible, denied (window expired), ambiguous boundary, reseller block, route-to-CS for higher tiers, human escalation for missing data.

The compliance-sensitive logic — the part where being wrong has real consequences — is rule-based. The part that genuinely needs natural language understanding (is this even a refund request?) is the LLM's job. Hard wall between them.

**The contract.** When the classifier fails — rate limit, parse error, missing key — the ticket never reaches the eligibility tree. It escalates to a human. There's a build-breaking test (`TestNotARefundRequestBranch`) that enforces this. A model failure cannot become a customer-facing decision.

---

## 2. Build-breaking tests for things you cannot enforce on an LLM

**The decision.** Refund-template language is enforced by a test that scans every template for promise-phrases — "refund approved," "refund issued," "we have refunded." If anyone edits a template into making a promise the agent isn't authorized to make, the build fails.

**Why it matters.** The agent's contract is "never promise a refund, only qualify the request." That's not enforceable on an LLM — you can prompt-engineer toward it, evaluate against it, monitor for it, but you cannot guarantee it. You *can* enforce it on the templates, because templates are deterministic strings. So that's where the contract lives.

This is a more general pattern: identify the part of your AI system's contract that's actually deterministic and pin it down with tests, even when the surrounding behavior is probabilistic. Other examples in the same vein:

- `test_pipeline_ids.py` validates pipeline and stage IDs against the live API. Caught a stage-ID mismatch that would have routed tickets to the wrong queue silently.
- `test_filter_count_stays_under_hubspot_cap` — worst-case multi-pipeline scan must produce ≤ 18 filters. HubSpot rejects searches above the cap, and this test was added after a `VALIDATION_ERROR` bricked a cron run during multi-pipeline rollout.
- A conftest autouse fixture forces the Slack client to `None` in every test so pytest runs with a real `SLACK_BOT_TOKEN` in the environment can't leak test ticket IDs into the live ops channel.

The total suite is 770+ tests. The build-breaking ones are the load-bearing safety guarantees the LLM cannot make for itself.

---

## 3. Match-first incident detection — bounded LLM spend by design

**The decision.** Incidents are persistent entities in a Qdrant collection, not per-run snapshots. New tickets match into existing incidents by cosine similarity to the running centroid. The LLM is consulted only when a new cluster seeds, or when a cluster broadens enough that its summary needs a refresh.

**Why it matters.** The first-pass architecture re-ran Claude over the full window every cycle, recomputing severity and summary from scratch. That's expensive at scale and produces inconsistent labels across runs (the cluster looks the same to a human but Claude names it differently each time).

The persistent-entity rewrite changes the cost model:

- **Match into existing incident → no LLM call.** Pure vector lookup.
- **Seed a new cluster → one LLM call** (only when the pending pool accumulates ≥ 3 unmatched tickets that cluster tight enough).
- **Bounded relabels → `log_1.5(n)` calls over an incident's lifetime** via a relabel multiplier. An incident that grows from 3 → 30 tickets gets ~8 relabel calls total, not 30.

Cost is now a function of incident *churn*, not ticket *volume*. A queue with steady, well-clustered traffic costs a fraction of a queue with the same volume but more diverse failure modes — which is the right shape for the cost curve.

**Severity is arithmetic, not LLM.** Severity is a pure function of rolling `ticket_count` in the 48h window: medium (3–9), high (10–14), critical (15+). Same reasoning as the refund eligibility tree — when the answer is reproducibly derivable from inputs, the LLM has no business being in the loop.

---

## 4. Threshold tuning is an operational discipline, not a config setting

**The decision.** Retrieval thresholds and incident-match thresholds are tuned from live traffic, not guessed at. Each one has a documented reason for its current value and a story about how it got there.

**Why it matters.** Most RAG implementations pick a similarity threshold once, ship it, and never revisit. The threshold-as-config pattern misses that retrieval quality has a direct, measurable relationship to the data you're actually retrieving against — which changes as your KB grows and as your traffic mix shifts.

**Examples from the system:**

| Collection | Threshold | History |
|---|---|---|
| `kb_jira` (known issues) | **0.70** | Started at 0.85. Tuned down after live calibration showed the threshold was too strict for the real distribution — genuine matches were being missed. |
| `kb_internal` (workarounds) | **0.60** | Workarounds are paraphrased and partial; broader net is correct here. |
| `kb_known_issues` (curated) | **0.70** | Matches Jira tier — a wrong issue reference undermines agent credibility. |
| `kb_<product>` (public docs) | **0.40** | Docs are written from many angles for the same concept; cast wide. |
| `kb_incidents` (incident match) | **0.75** | Started at 0.85. Lowered after a 12-ticket regression embedded at 0.75–0.85 and would have been missed. Earlier clusters tended to be verbatim error-message repetition; broader traffic is more prose-like. |

The 0.85 → 0.75 incident-threshold tune is the one I'd point at hardest. It only happened because the system was producing audit logs I could go back through and ask: how did this 12-ticket cluster behave under the old threshold? The answer was "it would have been three independent incidents," and that's the gap between catching an outage in hours vs. days.

**The operational discipline:** every threshold has a date, a reason, and a way to evaluate whether it's still right. The thresholds are versioned in code with comments. A six-month-old threshold value with no commentary is a code smell.

---

## 5. Module isolation — failure in one module does not block another

**The decision.** The main pipeline (support triage) and the lead gen agent are independent modules invoked side-by-side by the same scheduler. They scan disjoint sets of HubSpot pipelines and have separate state files, dedup, and audit logs. A failure in one cannot block the other.

**Why it matters.** The two modules share infrastructure (HubSpot client, Qdrant, Slack notifier) but have completely different operational profiles. The main pipeline runs against high-volume consumer queues with strict response-time expectations. Lead gen runs against lower-volume higher-tier queues with much looser timing — its job is to surface qualified leads, not to process every ticket on a deadline.

If the lead gen agent throws an unhandled exception (a malformed contact, a HubSpot API quirk in a rare property), the main pipeline keeps running. If the main pipeline hits a HubSpot rate limit, lead gen's slower cadence is unaffected.

Most "monorepo of agents" architectures share a single failure mode by accident. This one has a deliberate fault boundary at the module level — separate processes, separate state, separate logs.

---

## 6. Safety gates default to the safe state — zero-configuration is safe

**The decision.** Every side-effecting subsystem is gated by an env-var dry-run flag that defaults to `true`. Running the pipeline cannot touch a customer without someone explicitly opting in.

**Why it matters.** "Safe" is the boring default; you opt into live send by exporting a flag. If the dry-run lines are absent from the shell config, the Python defaults take over — all three resolve to `True` and nothing goes live. This means a fresh checkout, a misconfigured deploy, or a partial rollback all fail in the same direction: nothing happens to a customer.

**The kill-switch hierarchy.** Three levels, fastest first:

1. **Comment the export lines in the shell config** — next cron cycle (≤ 3h) reads the file fresh and reverts to safe defaults. No code change, no deploy.
2. **`unset` the env var in the current shell** — immediate, but only affects the current process.
3. **`pkill -f "main.py --live"`** — stops a run already in progress.

The kill-switch prevents *future* writes. Anything already written stays written; there's no rollback. That's by design — the system's job is not to be undoable, it's to be controllable.

---

## 7. Audit log on every decision — replay is a first-class capability

**The decision.** Every decision the system makes appends to a JSONL audit log: routing classifications, agent actions, escalations, refund eligibility outcomes, lead-gen qualifications. There are three separate logs (main pipeline, refund agent, lead gen) so a failure or a misroute in one subsystem doesn't pollute the others.

**Why it matters.** Two reasons that matter most operationally:

1. **Debugging is a database query, not a re-run.** When a customer escalates a misroute, I can grep the audit log and see exactly what the system saw, what rule fired, what the LLM returned, and what action was taken. No "let me try to reproduce it" stage.
2. **Threshold tuning is data-driven** (see Section 4). The 0.85 → 0.75 incident-threshold change happened because the audit log made it possible to ask "what would this incident have looked like under the old threshold?" — retrospectively, on real data.

Audit logs are also the foundation for the future eval/DPO pipeline. The "this was wrong" feedback button in the dashboard writes structured corrections back into a feedback corpus, keyed against the audit log entry. That's a v2+ item, but the data is being captured today.

---

## What this approach buys you

A theme runs through these decisions: **the LLM is one component, not the system.** The system is the rules layer, the retrieval layer, the agent layer, the QA layer, the audit layer, and the human-in-the-loop layer working together. The LLM does the part that genuinely requires natural language understanding. Everything else is engineering.

That separation is what makes the system durable. When Claude releases a new model, swapping it in is a config change. When HubSpot adds a new field, the rules layer extends. When a new failure mode emerges, the audit log shows it. The architecture absorbs change because the change surface is bounded.

Most production AI failures I've seen come from systems where the LLM is asked to do too much — eligibility decisions, routing logic, compliance enforcement — because the team didn't build the deterministic scaffolding around it. The scaffolding is the work. The LLM is the easy part.
