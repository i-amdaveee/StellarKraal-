# StellarKraal API — Endpoint Reference

This document provides a human-readable reference for all endpoints defined in
`openapi.json`. It mirrors the descriptions in the spec and is intended for
quick navigation during development and code review.

## Authentication (`auth`)

| Method | Path | Summary |
|--------|------|---------|
| GET | `/auth/challenge` | Get a one-time challenge |
| POST | `/auth/login` | Login with Stellar signature |
| POST | `/auth/refresh` | Rotate refresh token |

## Health (`health`)

| Method | Path | Summary |
|--------|------|---------|
| GET | `/health` | Service health check |
| GET | `/health/live` | Kubernetes liveness probe |
| GET | `/health/ready` | Kubernetes readiness probe |
| GET | `/health/deep` | Deep infrastructure health check |
| GET | `/health/{loanId}` | Get loan health factor |

## Collateral (`collateral`)

| Method | Path | Summary |
|--------|------|---------|
| POST | `/collateral/register` | Register livestock collateral (on-chain) |
| POST | `/collateral/register/batch` | Batch register livestock collateral (on-chain) |
| POST | `/collateral` | Create collateral record (DB) |
| GET | `/collateral` | List collateral records |
| GET | `/collateral/{id}` | Get collateral by ID |
| PUT | `/collateral/{id}/appraise` | Update appraised value |
| PATCH | `/collateral/{id}` | Partially update collateral fields |
| DELETE | `/collateral/{id}` | Soft-delete a collateral record |
| PATCH | `/collateral/{id}/restore` | Restore soft-deleted collateral |

## Loans (`loan`)

| Method | Path | Summary |
|--------|------|---------|
| POST | `/loan/request` | Request a new loan |
| POST | `/loan/repay` | Repay a loan |
| POST | `/loan/liquidate` | Liquidate an undercollateralised loan |
| POST | `/loan/repayment-preview` | Preview repayment breakdown |
| GET | `/loan/{id}` | Fetch a loan record |
| DELETE | `/loan/{id}` | Soft-delete a loan record |
| GET | `/loans` | List loans (paginated) |
| GET | `/loans/summary` | Get borrower loan portfolio summary |

## Oracle (`oracle`)

| Method | Path | Summary |
|--------|------|---------|
| POST | `/oracle/price-update` | Invalidate appraisal cache |

## Webhooks (`webhooks`)

| Method | Path | Summary |
|--------|------|---------|
| POST | `/webhooks` | Register a webhook |
| DELETE | `/webhooks/{id}` | Delete a webhook |

## Admin (`admin`)

| Method | Path | Summary |
|--------|------|---------|
| POST | `/alerts/webhook` | AWS SNS alert receiver |
| GET | `/admin/webhooks` | List registered webhooks |
| GET | `/admin/webhooks/logs` | List webhook delivery logs |
| GET | `/admin/migrations/status` | Migration status |
| GET | `/admin/deleted/collateral` | List soft-deleted collateral |
| POST | `/admin/restore/collateral/{id}` | Restore soft-deleted collateral |
| GET | `/admin/deleted/loans` | List soft-deleted loans |
| POST | `/admin/restore/loans/{id}` | Restore soft-deleted loan |
| GET | `/admin/users` | List borrowers |
| GET | `/admin/moderation-queue` | Pending-review loans |
| GET | `/admin/statistics` | Aggregate platform statistics |

## Transactions (`transactions`)

| Method | Path | Summary |
|--------|------|---------|
| GET | `/transactions` | List transactions |
| GET | `/transactions/{id}` | Get transaction by ID |
| GET | `/transactions/{hash}/status` | Poll transaction status on-chain |

## User Settings (`settings`)

| Method | Path | Summary |
|--------|------|---------|
| GET | `/settings/{wallet}` | Get user settings |
| PUT | `/settings/{wallet}` | Update user settings |

---

### Error Response Envelope

All error responses follow the `{ error, code, details? }` JSON envelope
defined by the `ErrorResponse` schema in `openapi.json`. Component-level error
responses (`ValidationError`, `Unauthorized`, `NotFound`, `RateLimited`,
`InternalError`) include example payloads for use in Swagger UI and
client SDK generation.

### Documentation Source of Truth

- **OpenAPI spec**: [`openapi.json`](../openapi.json)
- **Swagger UI**: available at the server root when `NODE_ENV !== 'production'`
- **This file**: mirrors the spec for quick reference; always validate against
  the canonical `openapi.json`.

---

## Loan State Machine & On-Chain Event Mapping

The loan lifecycle is driven by the Soroban loan contract. Every state
transition emits exactly one contract event; off-chain listeners should treat
the event stream as the authoritative source of truth and reconcile the DB
record against it.

### States

| State | Description | Entry conditions | Exit conditions |
|-------|-------------|------------------|-----------------|
| `Requested` | Borrower has submitted a loan request; awaiting review/approval. | `request_loan` succeeds (collateral registered & appraised). | Approved → `Approved`; rejected/cancelled → `Cancelled`; expired → `Expired`. |
| `Approved` | Loan approved and funds reserved; awaiting disbursement. | `approve_loan` succeeds on a `Requested` loan. | Disbursed → `Active`; cancelled → `Cancelled`; expired → `Expired`. |
| `Active` | Principal disbursed; borrower owes repayment. | `disburse_loan` succeeds on an `Approved` loan. | Fully repaid → `Repaid`; overdue → `Defaulted`; liquidated → `Liquidated`. |
| `Repaid` | Loan fully repaid (terminal). | `repay_loan` covers outstanding principal + interest. | — (terminal) |
| `Defaulted` | Loan past due and not repaid (terminal unless liquidated). | Grace period elapses while `Active` and unpaid. | Liquidated → `Liquidated`. |
| `Liquidated` | Collateral seized to cover the debt (terminal). | `liquidate_loan` succeeds on `Active`/`Defaulted` undercollateralised loan. | — (terminal) |
| `Cancelled` | Loan cancelled before disbursement (terminal). | `cancel_loan` on `Requested`/`Approved`. | — (terminal) |
| `Expired` | Approval/request expired before next step (terminal). | Expiry timestamp passes in `Requested`/`Approved`. | — (terminal) |

### Transition → Event Mapping

| From | To | Contract event | Payload |
|------|----|----------------|---------|
| — | `Requested` | `loan_requested` | `{ loan_id, borrower, collateral_id, principal, interest_rate, due_at }` |
| `Requested` | `Approved` | `loan_approved` | `{ loan_id, approver, approved_at }` |
| `Approved` | `Active` | `loan_disbursed` | `{ loan_id, borrower, amount, disbursed_at }` |
| `Active` | `Repaid` | `loan_repaid` | `{ loan_id, payer, amount, remaining, repaid_at }` |
| `Active` | `Defaulted` | `loan_defaulted` | `{ loan_id, borrower, overdue_since }` |
| `Active`/`Defaulted` | `Liquidated` | `loan_liquidated` | `{ loan_id, liquidator, collateral_id, proceeds, liquidated_at }` |
| `Requested`/`Approved` | `Cancelled` | `loan_cancelled` | `{ loan_id, cancelled_by, reason }` |
| `Requested`/`Approved` | `Expired` | `loan_expired` | `{ loan_id, expired_at }` |

All events are emitted with topics `("loan", <event_name>, loan_id)` so
listeners can filter by loan. Amounts are in stroops (i128).

### Invalid Transitions & Errors

Attempting a transition that is not permitted for the current state reverts
the transaction and returns a contract error (surfaced by the API as a
`409 Conflict` with the corresponding `code`).

| Attempted transition | Current state | Contract error | API `code` |
|----------------------|---------------|----------------|------------|
| `approve_loan` | not `Requested` | `InvalidState` | `LOAN_INVALID_STATE` |
| `disburse_loan` | not `Approved` | `InvalidState` | `LOAN_INVALID_STATE` |
| `repay_loan` | not `Active` | `InvalidState` | `LOAN_INVALID_STATE` |
| `liquidate_loan` | not `Active`/`Defaulted` | `InvalidState` | `LOAN_INVALID_STATE` |
| `cancel_loan` | not `Requested`/`Approved` | `InvalidState` | `LOAN_INVALID_STATE` |
| any transition | terminal state | `TerminalState` | `LOAN_TERMINAL_STATE` |
| `repay_loan` | `Active`, amount < owed | `InsufficientRepayment` | `LOAN_INSUFFICIENT_REPAYMENT` |
| `liquidate_loan` | `Active`, health factor ≥ 1 | `NotLiquidatable` | `LOAN_NOT_LIQUIDATABLE` |

### Lifecycle Timing Diagram

```
borrower        contract                 listener (off-chain)
   |               |                          |
   | request_loan  |                          |
   |-------------->|                          |
   |               | emit loan_requested      |
   |               |------------------------->|  state: Requested
   |               |                          |
   |               | approve_loan             |
   |               | emit loan_approved       |
   |               |------------------------->|  state: Approved
   |               |                          |
   |               | disburse_loan            |
   |               | emit loan_disbursed      |
   |               |------------------------->|  state: Active
   |               |                          |
   | repay_loan    |                          |
   |-------------->|                          |
   |               | emit loan_repaid         |
   |               |------------------------->|  state: Repaid (terminal)
```

Alternate branches from `Active`: `loan_defaulted` (→ `Defaulted`) followed by
`loan_liquidated` (→ `Liquidated`), or `loan_cancelled`/`loan_expired` from
`Requested`/`Approved`.
