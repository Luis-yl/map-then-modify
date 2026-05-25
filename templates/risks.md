# Architecture Risks

Last updated: `{{YYYY-MM-DD HH:MM}}`

Risks are not failures. **Hidden risks are failures.** Every risk here should have an explicit mitigation or be accepted with reason.

## Active Risks

| ID | Area | Risk | Severity | Confidence | Mitigation | Owner module |
| --- | --- | --- | --- | --- | --- | --- |
| `R{{####}}` | {{area / module}} | {{what could go wrong}} | {{low|medium|high|critical}} | {{confidence in this risk assessment}} | {{mitigation strategy or "accepted"}} | `{{M####-slug}}` |

## Stale Documentation

Modules whose docs are out of sync with code.

| Module / area | Staleness signal | Required refresh |
| --- | --- | --- |
| `{{M####-slug or area}}` | {{anchor moved / file renamed / test changed / config changed}} | {{refresh action}} |

## Verification Gaps

Modules with insufficient tests or no executable verification.

| Module | Missing verification | Risk | Suggested command or test |
| --- | --- | --- | --- |
| `{{M####-slug}}` | {{what's not tested}} | {{risk if regressed}} | `{{command_or_test}}` |

## External Or Unavailable Systems

| System | Used by | Availability | Risk |
| --- | --- | --- | --- |
| {{system name — e.g., mainnet RPC, oracle X, partner API}} | `{{M####-slug}}` | {{available|unavailable_in_dev|unknown}} | {{risk of relying on it}} |

## Generated Code Without Known Source

When you cannot find the generator. These are landmines.

| Generated artifact | Where used | Status |
| --- | --- | --- |
| `{{path}}` | `{{M####-slug}}` | {{generator missing / candidate found but unverified}} |

## Resolved Risks

| ID | Resolved in | Resolution |
| --- | --- | --- |
| `R{{####}}` | `{{dev-plan or commit or date}}` | {{what changed}} |
