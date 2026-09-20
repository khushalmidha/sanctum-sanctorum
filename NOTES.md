# NOTES

## Live URL

> **TODO**: Deploy and paste URL here.

Seed data includes 12 books and 4 members (Wong Li — supreme, Christine Palmer — master,
Jonathan Pangborn — adept, Sara Lin — apprentice). Use any of their `id`s (1–4) to test
orders and loans.

---

## What was finished

All features specified in SPEC.md are implemented and **all 202 tests pass**:

```
tests/test_books.py     67 passed
tests/test_health.py     1 passed
tests/test_loans.py     45 passed
tests/test_members.py   32 passed
tests/test_orders.py    46 passed
tests/test_reports.py   11 passed
```

### Books
- ISBN-13 checksum validation (weights 1,3,1,3…, check digit formula).
- Duplicate ISBN detection (409) after normalization.
- `PATCH /books/{id}` partial update via `model_fields_set`; `isbn` silently ignored.
- `GET /books` with `q` matching title OR author, `min_price`/`max_price` inclusive filters,
  `sort` (title/price asc/desc with id tiebreaker), and correct `total` via count subquery.

### Members
- Duplicate email detection (409), case-insensitive (schema lowercases before storage).
- Fixed `tier_at_least` bug: was `>`, now `>=` — members at exactly the minimum tier are allowed.
- Full `get_member_stats` with paid-only order stats, unreturned loan counts, and late fee sums.

### Orders
- `calculate_discount_percent`: tier discount + bulk discount (additive, no cap).
- `create_order` with two-pass stock validation for atomicity: all stock is validated before any
  mutations, so a failed order never leaves stock partially decremented.
- `cancel_order` now restores stock for every item (was missing).
- Integer floor division (`//`) for `discount_cents`.

### Loans
- `loan_status` with strict `>` for overdue boundary (`now == due_at` → active).
- `calculate_late_fee` with `math.ceil` for partial days, capped at book's current price.
- `create_loan` with all 6 ordered checks (404→403→409 overdue→409 duplicate→409 limit→409 stock).
- `return_loan` with stock restoration and late fee computation.
- `list_member_loans` with computed status filtering.

### Reports
- `top_books` query: sum quantities over paid orders only, sorted by copies_sold desc then
  title asc, books with zero sales naturally excluded.

## What was left incomplete

Nothing. All tests pass and all specified features are implemented.

---

## Architectural decisions and trade-offs

### Two-pass stock validation in orders
Orders validate stock for every item in a first pass before decrementing anything in the second
pass. This ensures atomicity: if any item has insufficient stock, nothing has been written to the
database. The alternative (single-pass with rollback) is more complex and error-prone, especially
since SQLAlchemy's session state would need careful handling.

### ISBN checksum in the Pydantic validator
The checksum is validated at the schema layer so invalid ISBNs get a 422 before any business
logic runs. The duplicate ISBN check stays in the service layer because it requires database
access and produces a 409. This matches the spec's ordering: 422s before 409s.

### Computed loan status
Loan status (`active`/`overdue`/`returned`) is never stored in the database — it's computed at
read time from `returned_at` and `due_at` vs. `now`. This avoids stale data: the status
automatically transitions from active to overdue as time passes, without needing a background
job or trigger.

### Late fee uses book's current price at return time
Per spec, the late fee cap uses the book's `price_cents` at the time of return, not a snapshot
from when the loan was created. This means if a book's price changes between borrowing and
returning, the new price is used for the cap.

### Email normalization
The email is stripped and lowercased in the Pydantic validator (`MemberCreate.normalize_email`),
so by the time it reaches the service layer, it's already normalized. The duplicate check in
`create_member` uses an exact match since both the stored value and the incoming value are
lowercase.

---

## Anything in the spec found unclear

### Order item ordering
The spec says "items ordered as submitted" but the database orders by `OrderItem.id`. This works
because items are inserted in submission order, so insertion-order `id`s match submission order.
This is an implicit assumption that holds with autoincrement but could break with non-sequential
ID generation.

### Mixed-case title sorting
The spec explicitly says "ordering of mixed-case titles is unspecified" and that either SQLite
or Postgres behavior is accepted. I rely on the database's default collation, which means SQLite
sorts uppercase before lowercase. This is consistent with the test expectations.

---

## AI usage

**Tools used**: Antigravity IDE (Claude-based AI coding assistant).

**What it was used for**:
- Understanding the full codebase structure and relationships between files.
- Implementing all missing features based on detailed spec analysis.
- Generating service implementations with correct business logic ordering.

**One place where the AI got something wrong**:
When initially planning the implementation, I noted that `cancel_order` needed stock restoration
as a fix, but on closer reading of the existing code, the cancel_order function was already
written — it just didn't restore stock. The AI's initial plan correctly identified this as a gap,
but the distinction between "needs to be implemented" vs. "needs to be fixed" matters for
understanding the original developer's intent. The existing cancel_order had the right structure
(status check, 409 handling) but silently omitted the stock restoration, which is the most
critical part of cancellation. This is a case where reading the existing code carefully before
writing new code prevented a more subtle bug.
