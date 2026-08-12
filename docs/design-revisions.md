# Design revisions (approach)

Concrete changes adopted after a Muratori / Ousterhout-style review. Status: **accepted into docs direction**.

## Revisions

| # | Revision | Action |
|---|---|---|
| 1 | Explicit **v1 slice** cut line | [v1-slice.md](./v1-slice.md) |
| 2 | Fixed Hypeluxe `ResolvedPolicy` — no multi-scope policy engine in v1 | v1-slice + platform-engineering note |
| 3 | **Split authority:** ops local-first; billing/client funds online-authoritative | v1-slice + architecture |
| 4 | Money **facades:** Billing / ClientFunds / DayOfCash | v1-slice + budget-and-money |
| 5 | Core = pure decisions; Effect at shell; `runCommand` wraps audit/authz | platform-engineering + architecture |
| 6 | Journey-first CD smoke (3 journeys) | v1-slice + roadmap |
| 7 | Entity diet (defer TransferPlan entity, Note CRDT, etc.) | v1-slice |
| 8 | Complexity rule on new knobs/entities | v1-slice + decisions |
| 9 | **Drawdowns remain in v1** | Planner control: cleared milestones + actual costs + budget — not deferred |

## Drawdowns (planner control loop)

Drawdowns are not optional polish. They let planners:

1. Spend only against **cleared** client payments (milestones acknowledged)  
2. Allocate only up to vendor **actual** (invoice received)  
3. See burn vs **budget** so the event does not run ahead of collections  

v1 includes `ClientFunds` (clear + drawdown) alongside `Billing` and budget/cost.

## What we did *not* abandon

- Event-agnostic north star  
- Hypeluxe as first tenant  
- FSMs, audit, SSO/passwordless, PIPEDA+GDPR  
- Manual flows before payment-processor automation  
- Day-of cash for coordinators without full ledger access  
