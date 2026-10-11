---
date: 2026-10-11
topic: ledger-demo-ux-brief
---

# Ledger Demo: UX Brief for Claude Design

## Summary

Design a self-serve web app for engineers who are new to our BaaS credit ledger. They pick one of five real API commands, read its payload, run it, and see the whole effect on one canvas: **API Command → Ledger → Account → Event Stream**. The Account and Event Stream nodes show their changes directly on the canvas. A bottom inspector gives depth on whichever node is selected, and Rewind undoes the last run.

---

## Problem Frame

New engineers learn the platform from API docs, by shadowing colleagues, by reading code and by firing commands in Postman. Each source shows one component in detail and none shows the end-to-end behaviour. Engineers get stuck on component internals before they understand what a single command does to an account and what it publishes downstream. As a result, their first app on the platform is built without a working mental model of command → ledger → account state → events.

---

## Audience

- A1. **New engineer, onboarding (primary).** Technically fluent and used to Postman and JSON, but new to this ledger's vocabulary. Uses the demo alone, without a presenter. The goal is to leave able to predict which balances, calculated fields and events a given command will touch before building against the platform.

There is no secondary audience in v1. Where a choice trades plain-language explanation against technical fidelity, choose technical fidelity: show real field names and real payloads.

---

## Key Decisions

- **End-to-end first, component detail on demand.** One run must show the full chain without any clicking. The diagram and inspector are where engineers go deeper afterwards, not the only way to see the result.
- **Account and Event Stream nodes are live mini-dashboards.** They render balance deltas and emitted events directly on the canvas. This was chosen over a lean diagram that hides results in the inspector, and over a trace-first layout that demotes the diagram.
- **Four components, in this order:** API Command → Ledger → Account → Event Stream. Don't add nodes for internal services.
- **Read-only payloads, fixed command set.** Engineers pick from the five defined commands and cannot edit payloads in v1.
- **Rewind is a demo undo, not a ledger operation.** It restores the account to its state before the last run and removes that run's events. The UI must say this plainly so engineers don't infer that real ledger entries can be deleted.
- **Runs stack.** Each run applies to the account as the previous run left it, so engineers can chain commands (e.g. PostTransaction then ApplyPayment). Rewind steps back one run at a time.
- **The Ledger is not a black box.** Its inspector shows the financial transaction and postings it wrote for the run.

---

## Platform Vocabulary

Use these names verbatim in the UI.

**Commands (5):** `PostTransaction`, `ApplyPayment`, `ApplyCashAdvance`, `PostBalanceTransferRequest`, `PostFeeAction`

**Account attributes (7):** `cashInterestRate`, `balanceTransferInterestRate`, `purchaseInterestRate`, `accountOpenDate`, `currentDate`, `liveTestAccount`, `provisionDate`

**Balances (9):** `cashAdvanceBalance`, `cashReceived`, `genericFeeBalance`, `interestBalance`, `interestPaid`, `lateFeeBalance`, `lateFeePaid`, `purchaseBalance`, `promotionalBalance`

**Calculated fields (2):** `openToBuy`, `totalHolds`

**Egress event types (8):** Account created, Account status, Account billing, Payment made, Balance transfer posted, Credit balance refund trigger, Financial transaction, Maintenance transaction

**Assumed command effects (for sample data only; to be confirmed):**

| Command | Balances likely touched | Calculated fields | Events likely emitted |
|---|---|---|---|
| `PostTransaction` | `purchaseBalance` ↑ | `openToBuy` ↓ | Financial transaction |
| `ApplyPayment` | `cashReceived` ↑; one or more balances ↓ per allocation | `openToBuy` ↑ | Payment made, Financial transaction; Credit balance refund trigger if overpaid |
| `ApplyCashAdvance` | `cashAdvanceBalance` ↑ | `openToBuy` ↓ | Financial transaction |
| `PostBalanceTransferRequest` | `promotionalBalance` ↑ | `openToBuy` ↓ | Balance transfer posted, Financial transaction |
| `PostFeeAction` | `genericFeeBalance` or `lateFeeBalance` ↑ | `openToBuy` ↓ | Maintenance transaction |

---

## Layout

```text
┌──────────────────────────────────────────────────────────────────────┐
│ Command bar: [command picker ▾]  [Run]  [Rewind]   run history chips │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────┐    ┌────────┐    ┌──────────────────┐    ┌─────────┐  │
│   │   API    │ ─▶ │ Ledger │ ─▶ │ Account          │ ─▶ │ Event   │  │
│   │ Command  │    │        │    │ attrs · 9 bals · │    │ Stream  │  │
│   │          │    │        │    │ 2 calc (deltas)  │    │ (chips) │  │
│   └──────────┘    └────────┘    └──────────────────┘    └─────────┘  │
│                         React Flow canvas                            │
├──────────────────────────────────────────────────────────────────────┤
│ Inspector (resizable, collapsible): selected node · tabs             │
└──────────────────────────────────────────────────────────────────────┘
```

Directional only. It shows the regions and their order; proportions, styling and component choices are for Claude Design to decide.

---

## Requirements

**Command bar**
- R1. Engineers choose a command from the five listed. Each option shows its name and a one-line description of what it does.
- R2. Before running, the selected command's payload is visible as read-only, syntax-highlighted JSON, alongside a short "expected effect" preview (which balances, calculated fields and events to watch).
- R3. A single Run action executes the selected command against the account in its current state.
- R4. A Rewind action undoes the most recent run. It is disabled when there is nothing to rewind, and it is labelled or tooltipped as a demo control, not a ledger reversal.
- R5. A compact run history shows the runs so far, in order, as the stack Rewind walks back through.

**Canvas and run playback**
- R6. The canvas shows four nodes in a left-to-right flow: API Command, Ledger, Account, Event Stream.
- R7. When a command runs, the flow plays in visible order, one hop at a time (Command → Ledger → Account → Event Stream), so engineers see the sequence and not just the end state.
- R8. Every node has a distinct visual state for idle, active (currently processing), done for this run, and failed.
- R9. Before the first run, the canvas explains itself: the four nodes, labelled edges and a one-line prompt to pick and run a command.
- R10. After a run, a single run summary line states the outcome, e.g. `PostTransaction → 1 transaction · 2 of 9 balances changed · openToBuy −120.00 · 1 event`.

**API Command node**
- R11. Shows the command name and a short identifier for the current run. The full payload is in the inspector.

**Ledger node**
- R12. Shows that a financial transaction was written for the run, with a count of postings.
- R13. Its inspector shows the transaction and its postings, so engineers see the record the ledger wrote between the command and the balance change.

**Account node (mini-dashboard)**
- R14. Shows all 9 balances and both calculated fields as compact, fixed-position tiles that never reflow between runs.
- R15. After a run, changed tiles show before → after and a signed delta. Unchanged tiles visibly recede.
- R16. Calculated fields (`openToBuy`, `totalHolds`) are styled differently from stored balances, so engineers can tell computed values from stored ones at a glance.
- R17. Account attributes are reachable from the node (e.g. a compact row or a count with a link to the inspector) and use the same changed/unchanged treatment.

**Event Stream node (mini-dashboard)**
- R18. Shows each event emitted by the run as a typed chip, using the event-type names listed above, in emission order.
- R19. Events from earlier runs stay visible but de-emphasised, and grouped by run.

**Inspector**
- R20. A bottom panel shows details for the selected node. It can be resized and collapsed.
- R21. Every node uses the same tab structure (e.g. Overview · Data · Changes · Events) so engineers learn the panel once. Tabs with nothing to show for a node are hidden or marked as empty.
- R22. Content per node:
  - API Command: full payload.
  - Ledger: transaction and postings.
  - Account: all attributes, balances and calculated fields with before / after / delta, plus the formula for each calculated field with its current input values.
  - Event Stream: full event bodies for the selected run.
- R23. Selecting a changed balance or calculated field highlights the event(s) and posting(s) it relates to, and the reverse. This keeps the causal thread across surfaces.
- R24. During playback, the inspector follows the active node unless the engineer has pinned a node.

**Rejected runs**
- R25. When a command is rejected (e.g. it would exceed `openToBuy`), the flow stops at the node that rejected it, which takes the failed state. The Account shows no changes and the inspector explains the reason.

**Design system**
- R26. Built from the shadcn/ui component set already available to us, with React Flow for the canvas. Custom node bodies use shadcn primitives and tokens so the canvas and the chrome around it read as one system.
- R27. Supports light and dark themes, and stays readable on a laptop screen without zooming.

---

## Acceptance Examples

- AE1. **Covers R3, R7, R15, R18.** Given a fresh account, when the engineer runs `PostTransaction`, the flow plays Command → Ledger → Account → Event Stream. `purchaseBalance` and `openToBuy` show before → after with deltas, the other balances recede, and a Financial transaction chip appears.
- AE2. **Covers R5, Runs stack.** Given a completed `PostTransaction`, when the engineer runs `ApplyPayment`, the before values match the after values of the previous run. Run history shows two entries.
- AE3. **Covers R4, R5.** Given two completed runs, when the engineer clicks Rewind, the account returns to its state after run one, run two's events disappear, and run history shows one entry. Rewinding again returns to the fresh account and disables Rewind.
- AE4. **Covers R23.** Given a completed `ApplyPayment`, when the engineer selects the `cashReceived` tile, the Payment made event and the related posting are highlighted. Selecting the Payment made chip highlights `cashReceived` back.
- AE5. **Covers R25.** Given an account with low `openToBuy`, when the engineer runs `ApplyCashAdvance` for more than `openToBuy`, the flow stops at the rejecting node, which shows the failed state. No balance changes, and the inspector states the reason.

---

## Success Criteria

- Within about 3 seconds of a run finishing, an engineer can name which balances and calculated fields changed and which events were emitted, without clicking anything.
- After running all five commands, an engineer can predict the balances and events touched by a command before running it.
- Claude Design can produce a first design from this brief without inventing component names, field names or layout regions.

---

## Explore Freely

These are left open to Claude Design: the visual style of nodes and edges, how deltas are shown (tiles, table rows, T-account style), motion and timing of playback, information density and typography, the exact tab names in the inspector, and empty, loading and failed states.

---

## Scope Boundaries

**Deferred for later**
- Downstream listeners or consumers reacting to events.
- Editing payloads or amounts.
- Compensating reversal commands (real ledger reversals with their own postings and events).
- "Copy as cURL" or a Postman export of the payload.

**Outside this product's identity**
- A presenter or sales mode, captions for non-technical audiences.
- Volume and throughput views.
- Any connection to a live environment; the demo runs on simulated data.

---

## Outstanding Questions

**Resolve before design starts**
- `openToBuy` needs a credit limit, and no credit-limit attribute is listed. What is the field, and what is the exact formula?
- `totalHolds` implies holds exist, but no hold balance is listed. What feeds it, and which commands create or release holds?
- Confirm or correct the assumed command effects table, especially payment allocation order for `ApplyPayment`.

**Deferred to design or build**
- Example payloads for each command and example bodies for each event type. Until they arrive, use realistic sample data.
- Whether `currentDate` should move (e.g. a "skip to next cycle" control to show interest or late fees). Not in v1 unless trivial.
- What `liveTestAccount` and `provisionDate` mean for the demo, and whether any command changes them.
- Which commands should have a built-in rejected path (AE5 assumes `ApplyCashAdvance` over `openToBuy`).

---

## Sources

- Modern Treasury Ledgers: pending / posted / available balances; entries returning resulting balances (docs.moderntreasury.com/ledgers/docs/ledger-accounts-overview)
- TigerBeetle: stored totals vs derived values; rejecting transfers that would overdraw (docs.tigerbeetle.com/reference/account/)
- Inngest run traces and Redux DevTools: one run record shown through a detail pane with a consistent tab layout and diff view (inngest.com/docs/platform/monitor/traces)
- React Flow performance: animate a single element along the active edge in place of built-in animated edges (liambx.com/blog/tuning-edge-animations-reactflow-optimal-performance)
