# IntentGuard

> **Cryptography proves who authorised a purchase. IntentGuard proves the purchase matches what they meant.**

IntentGuard is a bank-side agent system that checks every purchase an AI shopping agent tries to make against what the customer actually asked for. It blocks manipulated purchases, asks the customer when a purchase is ambiguous, and builds the evidence chain when a purchase is disputed.

Built for the **BFSI theme of the Google Cloud AI Builder Cup 2026** on Google's agentic-commerce protocols (AP2, UCP, A2A), ADK and Gemini, deployed on Cloud Run and Firebase.

## Why IntentGuard?

AI agents are starting to buy things for people, and banks have no way yet to check that an agent's purchase is what the customer wanted.

- **Banks have asked for safeguards, but none are built.** In September 2026, six banks (ASB, Bank of America, Capital One, CommBank, ING, NatWest) published joint principles for trusted agentic commerce, naming compromised agents, impersonation and payment fraud as risks.
- **Signatures do not prove meaning.** A valid AP2 mandate shows *who* authorised a purchase, not that the cart matches the plain-language instruction.
- **LLM agents bring new attacks into payments.** Prompt injection and poisoned tool results are threats classical payment protocols never faced.

IntentGuard fills the gap: it turns the customer's plain-language delegation into a checkable policy, checks each agent purchase against it, catches manipulation, and produces the audit trail and dispute evidence the banks' principles call for.

**Design principle:** AP2 mandates are checked by deterministic code. Gemini judges whether a cart honours the intent and whether a merchant tried to manipulate the agent. The final **allow / step-up / block** decision is always taken by deterministic rules — never by an LLM alone.

## What it does

- **Delegate in plain language** — the customer writes a rule ("noise-cancelling headphones under AUD 200 from trusted stores by Friday"), sees it back as a structured policy, and confirms it.
- **Verify every agent purchase** — deterministic mandate checks first, then a semantic check of cart against intent.
- **Detect manipulation** — prompt injection, bait-and-switch, hidden add-ons and merchant impersonation.
- **Decide and explain** — every decision carries a plain-language explanation citing the policy clause and evidence.
- **Keep the customer in the loop** — ambiguous purchases trigger a step-up approval; unanswered step-ups expire as rejected.
- **Resolve disputes** — the Dispute Agent assembles an evidence pack and a draft liability finding for human review.
- **Trace everything** — one trace ID across every agent, tool and protocol call.

## Architecture

IntentGuard sits between the customer's shopping agent and the payment: the agent must ask IntentGuard over A2A before it pays, and only the Decision Engine's rules can let money move.

```text
 Customer app ──(rule)──► Policy Compiler ──► confirmed policy
                                                     │
 Shopping agent ◄──UCP──► Simulated merchants        │
      │                                              ▼
      └──A2A check request (AP2 mandates, cart, merchant, trace)──►
                                                     │
                                         Mandate Verifier (code)
                                          fail ─► BLOCK (no LLM call)
                                                     │
                                   ┌─────────────────┴─────────────────┐
                            Intent Matcher                  Manipulation Detector
                         (Gemini, per-line verdict)       (Gemini + Model Armor)
                                   └─────────────────┬─────────────────┘
                                                     ▼
                                         Decision Engine (rules)
                                    ALLOW · STEP-UP (ask customer) · BLOCK
                                                     │
                                  Firestore + BigQuery (case, evidence, trace)
                                                     │
                                         Dispute Agent (on request)
```

| Component | Runs on | Type | Responsibility |
|---|---|---|---|
| Customer app | Firebase Hosting, React, Firebase Auth | UI | Write and confirm rules, answer step-ups, view timeline, open disputes |
| Shopping agent | Cloud Run, ADK | Gemini agent | Shops with merchants for the customer; calls IntentGuard before paying |
| Simulated merchants | Cloud Run (3 services) | Simulators | Honest, injection and bait-and-switch behaviour, resettable |
| Policy Compiler | Cloud Run, ADK | Gemini | Plain-language rule to structured policy, ambiguities flagged |
| Mandate Verifier | Cloud Run | Deterministic code | AP2 signatures, expiry, replay, limits, merchant allowlist |
| Intent Matcher | Cloud Run, ADK | Gemini Flash | Per-line match, partial or mismatch, with reasons |
| Manipulation Detector | Cloud Run, ADK | Gemini + Model Armor | Injected instructions, swaps, hidden add-ons, impersonation |
| Decision Engine | Cloud Run | Deterministic rules | Allow, step-up or block; writes the explanation |
| Dispute Agent | Cloud Run, ADK, own A2A endpoint | Gemini | Evidence pack and draft liability finding |
| MCP tools | Cloud Run | MCP servers | Typed access to policies, cases, merchant registry, evidence |
| Data and platform | Firestore, BigQuery, Secret Manager, Cloud Logging, Cloud Trace | Managed services | State, analytics and evaluation, keys, tracing |

### Decision rules (first match wins)

| Condition | Decision |
|---|---|
| Mandate verification failed | Block |
| Manipulation detected with high confidence | Block |
| Any cart line is a mismatch | Block |
| Any cart line is partial, or manipulation suspected with low confidence | Step-up |
| Model output missing, malformed or low confidence | Step-up (fail safe, never silent allow) |
| All lines match, no manipulation | Allow |

### Protocols

| Protocol | Between | Purpose |
|---|---|---|
| **AP2** | Customer, shopping agent, IntentGuard | Signed mandates recording what the customer authorised and what the agent proposes to pay |
| **A2A** | Shopping agent ↔ IntentGuard; IntentGuard ↔ Dispute Agent | Check requests and decisions between independently deployed agents, discovered via Agent Cards |
| **UCP** (style) | Shopping agent ↔ simulated merchants | Catalogue, cart and checkout calls |
| **MCP** | IntentGuard agents ↔ tools | Policy store, case store, merchant registry and evidence tools with typed inputs |

### Example structured policy

```json
{
  "item": {"category": "headphones", "must_have": ["noise cancelling"], "exclude": ["refurbished"]},
  "budget": {"max_total": 200, "currency": "AUD", "recurring_allowed": false},
  "merchants": {"mode": "trusted_only", "allow": ["store-a.example"], "deny": []},
  "deadline": "2026-10-09",
  "step_up_when": ["variant_differs", "new_merchant"],
  "ambiguities": []
}
```

## Demo scenarios

| ID | Scenario | Expected outcome |
|---|---|---|
| S1 | Honest store, cart matches the rule (headphones, AUD 179, trusted store) | Allow, explanation logged |
| S2 | Same model in a different colour, within budget | Step-up; proceeds on approval |
| S3 | Injection store: hidden text tells the agent to add a premium bundle | Block, injection flagged |
| S4 | Bait-and-switch: price rises to AUD 289 and a subscription is added | Block, changed items and price listed |
| S5 | Impersonation: merchant domain imitates a trusted store | Block on allowlist and similarity check |
| S6 | Replayed or expired mandate | Block by the deterministic verifier, no LLM call |
| S7 | Customer disputes a past purchase | Evidence pack and draft liability finding |

## Evaluation

IntentGuard is evaluated on ~150 labelled synthetic scenarios (honest, borderline, prompt injection, bait-and-switch, impersonation, mandate faults). The evaluation set and runner live in [`eval/`](eval/); results are stored in BigQuery and shown on an evaluation dashboard. Only measured results are reported.

| Metric | Prototype target |
|---|---|
| Attack block rate (blocked or stepped up) | ≥ 95% |
| False block rate on honest purchases | ≤ 5% |
| Explanation accuracy (human-graded sample) | ≥ 90% |
| Mandate faults caught by the deterministic verifier | 100% |
| Injected tool/model failures ending in step-up or block | 100% |
| Latency (median, p95) and cost per check | Reported |

Latency target: under 5 s median for a full check; mandate-only blocks under 500 ms.

## Security and responsible AI

- **Untrusted content stays data** — merchant pages, product text, agent traces and dispute notes are passed to models as delimited data and screened by Model Armor.
- **Structured outputs only** — every agent returns schema-validated JSON; anything invalid becomes a step-up, never an allow.
- **Deterministic money decisions** — signatures, limits, allowlists and the final decision are code.
- **Least privilege** — separate service accounts per Cloud Run service; data access only through MCP tools that separate read from write.
- **Secrets** — signing keys in Secret Manager; never in prompts, logs, Agent Cards or the repo.
- **Customer control** — every active delegation is visible, pausable and revocable.
- **Human review** — dispute liability findings are drafts that require a human decision.

## Tech stack

- Google Cloud: Cloud Run, Firebase Hosting & Auth, Firestore, BigQuery, Secret Manager, Cloud Logging, Cloud Trace
- Gemini (Flash-class models) and Model Armor
- Google Agent Development Kit (ADK)
- Agentic-commerce protocols: AP2, A2A, UCP, MCP
- React frontend

## Status

🚧 **Hackathon prototype in active development** — submission targeted for 17 October 2026.

Setup and run instructions will be added as the prototype is implemented.

## Disclaimer

IntentGuard is a prototype. All merchants, mandates, customers and transactions are synthetic. It does not move real money, connect to a real bank or card network, or give legal rulings on liability.

## License

See [LICENSE](LICENSE).
