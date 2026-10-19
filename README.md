# Handsoff

Handsoff is a Soroban-based lending protocol with a React frontend for signing and submitting transactions.

## Loan State Machine

The loan lifecycle is modeled as a state machine. Each transition is emitted as a Soroban contract event so that off-chain listeners can reconstruct loan state deterministically.

### States

| State | Description | Entry conditions | Exit conditions |
| --- | --- | --- | --- |
| `Requested` | Borrower has created a loan request | `request_loan` succeeds | `approve_loan` (→ `Approved`), `cancel_loan` (→ `Cancelled`), `expire_loan` after deadline (→ `Expired`) |
| `Approved` | Lender has approved the request | `approve_loan` succeeds on a `Requested` loan | `fund_loan` (→ `Funded`), `cancel_loan` (→ `Cancelled`), `expire_loan` after deadline (→ `Expired`) |
| `Funded` | Principal has been disbursed to the borrower | `fund_loan` succeeds on an `Approved` loan | `repay_loan` (→ `Repaid`), `mark_default` after due date (→ `Defaulted`) |
| `Repaid` | Borrower has repaid principal plus interest | `repay_loan` succeeds on a `Funded` loan | Terminal |
| `Defaulted` | Loan passed its due date without repayment | `mark_default` succeeds on a `Funded` loan | Terminal |
| `Cancelled` | Loan was cancelled before funding | `cancel_loan` succeeds on a `Requested` or `Approved` loan | Terminal |
| `Expired` | Loan request expired before approval/funding | `expire_loan` succeeds after the deadline | Terminal |

### Transition → Event Mapping

Every successful transition emits exactly one Soroban contract event. The event topic is `("loan", <event_name>)` and the payload is the loan record plus transition-specific fields.

| Transition | Contract event | Payload |
| --- | --- | --- |
| `Requested` (create) | `loan_requested` | `{ loan_id, borrower, principal, interest_rate, deadline }` |
| `Requested` → `Approved` | `loan_approved` | `{ loan_id, lender, approved_at }` |
| `Approved` → `Funded` | `loan_funded` | `{ loan_id, lender, funded_at, amount }` |
| `Funded` → `Repaid` | `loan_repaid` | `{ loan_id, borrower, repaid_at, amount }` |
| `Funded` → `Defaulted` | `loan_defaulted` | `{ loan_id, defaulted_at }` |
| `Requested`/`Approved` → `Cancelled` | `loan_cancelled` | `{ loan_id, cancelled_by, cancelled_at }` |
| `Requested`/`Approved` → `Expired` | `loan_expired` | `{ loan_id, expired_at }` |

### Invalid Transitions

Attempting a transition that is not allowed from the current state fails with a typed contract error and emits no event.

| Attempted transition | Error |
| --- | --- |
| `approve_loan` on non-`Requested` loan | `LoanError::InvalidState` |
| `fund_loan` on non-`Approved` loan | `LoanError::InvalidState` |
| `repay_loan` on non-`Funded` loan | `LoanError::InvalidState` |
| `mark_default` before due date or on non-`Funded` loan | `LoanError::InvalidState` / `LoanError::NotYetDue` |
| `cancel_loan` on `Funded`/terminal loan | `LoanError::InvalidState` |
| `expire_loan` before deadline | `LoanError::NotYetExpired` |
| Any transition on a terminal loan | `LoanError::InvalidState` |

### Lifecycle Timing Diagram

```
Borrower            Contract                 Lender / Listener
   |                   |                          |
   |-- request_loan -->|                          |
   |                   |-- loan_requested ------->|
   |                   |                          |
   |                   |<------- approve_loan ----|
   |                   |-- loan_approved -------->|
   |                   |                          |
   |                   |<------- fund_loan -------|
   |                   |-- loan_funded ---------->|
   |                   |                          |
   |-- repay_loan ---->|                          |
   |                   |-- loan_repaid ---------->|
   |                   |                          |
```

If the borrower does not repay before the due date, the lender calls `mark_default`, which emits `loan_defaulted` and moves the loan to the terminal `Defaulted` state.

## Handsoff notes

<!-- handsoff-issue-1393 -->
- #1393: Add confirmation dialogs for destructive and high-stakes actions
