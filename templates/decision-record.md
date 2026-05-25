# Decision: {{decision_title}}

Date: `{{YYYY-MM-DD}}`
Status: `{{proposed|accepted|superseded}}`
Triggered by: {{`analysis-runs/...` | `dev-plans/...` | `self-healing/...`}}

## Context

{{Facts and constraints that forced this decision. What was the question or fork in the road? Be specific — name the modules, files, or invariants involved.}}

## Decision

{{The decision made, in one or two sentences. Imperative form preferred: "Treat the on-chain governance module as a leaf even though it spans three files, because all three files implement one cohesive responsibility."}}

## Alternatives Considered

| Alternative | Why not chosen |
| --- | --- |
| {{alternative}} | {{reason}} |

## Consequences

### Positive
- {{Positive consequence.}}

### Negative or risky
- {{Negative consequence or new risk introduced — link to `RISKS.md` entry if applicable.}}

## Evidence

| Evidence | Relevance |
| --- | --- |
| `{{path}}:{{lines}}` | {{why this evidence supports the decision}} |

## Supersedes / Superseded By

- Supersedes: {{`decisions/...` if any}}
- Superseded by: {{empty when this is the current decision}}
