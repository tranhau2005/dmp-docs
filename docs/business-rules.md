# Business rules

> **Owner-approved source:** `bao-cao.tex` (revision 2026-10-27). Precedence: `bao-cao.tex` (business rules) = `schema.dbml` (structure) > `docs/agent-guide.md` (conventions) > `kiro-system-architecture-handoff.md` (history), per review feedback §0.1. This file copies rules only; citations point to the source sections/tables. No personal records are reproduced.

## Permission matrix

| Function | ADMIN | DRIVER | CUSTOMER |
|---|---|---|---|
| Driver profile | All | Own profile | None |
| Edit contact information | All | Own profile | None |
| Change employment status | Yes | No | No |
| Submit and approve leave | Approve | Submit own | No |
| Manage customers | All | No | Own profile |
| Create/view bookings | All | Assigned bookings | Own bookings |
| Assign drivers | Yes | No | No |
| Manage timekeeping | All | View own | No |
| View/print income | All | Own | No |
| Calculate/finalize payroll | Yes | No | No |
| Permissions and reports | Yes | No | No |

The server checks the active account, role, and record scope for each operation. Drivers cannot change `driver_id` to access another driver's records; customers can only view their own bookings. No one may approve their own leave request. Driver-editable fields exclude driver code, employment status, and role. [Source: report, “Ma trận quyền nghiệp vụ”; §3.6.4.]

## Table rules (17 tables)

1. **`accounts`** — DRIVER accounts require `driver_id`; CUSTOMER accounts require `customer_id`; ADMIN accounts link to neither. A CHECK ensures the account links only to the profile matching its role. Customers need a CUSTOMER account to create and track bookings; ADMIN may enter a booking for a customer. Account lock status is independent of employment status. At a future RESIGN effective time, reject login and refresh when RESIGNED is effective and run a daily job to close `driver_vehicles` assignments (OWNER-APPROVED: Q6). [Source: report, §3.2, table `accounts`; §3.6.3; OWNER-APPROVED: Q6.]
2. **`areas`** — A driver's area reflects the current assignment. Historical area reporting is outside the design. [Source: report, §3.2, table `areas`.]
3. **`drivers`** — Date of birth precedes start date. Current employment status is derived from `employment_history`, not duplicated here. Drivers may edit only permitted contact fields. [Source: report, §3.2, table `drivers`.]
4. **`employment_history`** — A driver's intervals do not overlap; status coverage is continuous from start date; `valid_to > valid_from` when set; at most one interval is open. A time-limited approved leave splits WORKING and creates the return interval. A resignation creates open-ended RESIGNED. The source request must belong to the same driver. Approved request mapping: ANNUAL → ON_LEAVE; SICK → SICK; LONG → LONG_LEAVE; SUSPEND → SUSPENDED; RESIGN → RESIGNED. LOCKED is not an employment status. [Source: report, §3.2, table `employment_history`; §3.5.]
5. **`driver_licenses`** — `expires_on >= issued_on` when present. Validate license eligibility when assigning a vehicle; do not infer legal requirements from license-class codes. [Source: report, §3.2, table `driver_licenses`.]
6. **`vehicles`** — A vehicle may be assigned to different drivers in separate time intervals. [Source: report, §3.2, table `vehicles`.]
7. **`driver_vehicles`** — `valid_to > valid_from`; intervals cannot overlap for either a driver or a vehicle. [Source: report, §3.2, table `driver_vehicles`.]
8. **`customers`** — Do not delete a customer with bookings. Phone remains unique across the whole table, including archived records (OWNER-APPROVED: Q8). [Source: report, §3.2, table `customers`; OWNER-APPROVED: Q8.]
9. **`bookings`** — PENDING has no `driver_id`; ASSIGNED, IN_PROGRESS, and COMPLETED require one. `completed_at` and `fare_amount` are present iff status is COMPLETED; `cancelled_at` is present iff CANCELLED. Flow: PENDING → ASSIGNED → IN_PROGRESS → COMPLETED; PENDING or ASSIGNED may transition to CANCELLED; COMPLETED and CANCELLED are final. Cancellation creates no payable trip. Each completed booking is counted once in the local month of `completed_at`. The server validates state transitions. `fare_amount` is entered on completion; forecast fare would require a new column (OWNER-APPROVED: Q5). DRIVER may transition their own booking to IN_PROGRESS/COMPLETED; ADMIN may transition any booking (OWNER-APPROVED: Q1). Cancellation is supported only from PENDING/ASSIGNED (OWNER-APPROVED: Q2). [Source: report, §3.2, table `bookings`; §3.7.2; OWNER-APPROVED: Q1, Q2, Q5.]
10. **`leave_requests`** — PENDING has no decision fields; APPROVED/REJECTED require `decided_by` and `decided_at`. `end_date >= start_date`; RESIGN has no `end_date`. Do not approve overlapping leave periods. SUSPEND is applied directly by ADMIN (`source_request_id = NULL`); retain SUSPEND in `leave_requests` only for compatibility (OWNER-APPROVED: Q4). [Source: report, §3.2, table `leave_requests`; OWNER-APPROVED: Q4.]
11. **`leave_actions`** — Keep the action log immutable. Write the action and request decision in one transaction. SUBMIT actor is the request owner. Only ADMIN may APPROVE/REJECT, once, while PENDING; reject requires a comment. The decision flow is PENDING → APPROVED or REJECTED. Lock the request row and verify role and pending status. Approval updates the request, action log, and employment intervals in one transaction. Future requests create only future intervals; a leave ending on day 12 is effective through 00:00 on day 13. RESIGN creates no return interval, closes open vehicle assignments, and disables the account at its effective time. Block approval if it conflicts with an approved future request. A conflicting activity session must be resolved before payroll; do not silently delete it or create an automatic deduction. Withdrawal or post-approval edits need a separate workflow. [Source: report, §3.2, table `leave_actions`; §3.6.3.]
12. **`work_schedule`** — `planned_minutes` is 1–1440; `(driver_id, work_date)` is unique. No schedule means no expected activity under this model; absence of a session alone is not a violation. [Source: report, §3.2, table `work_schedule`.]
13. **`activity_sessions`** — A session belongs to one local calendar day in `Asia/Ho_Chi_Minh`; overnight work must be split, allowing an end exactly at 00:00 next day. `ended_at > started_at`; same-driver sessions cannot overlap; at most one session may be open. Open sessions cannot enter a finalized payroll period. Duration is measured in minutes; sum at source precision and round the total to two decimals. Count distinct positive-duration days, not sessions. `source_key` is a client-generated UUID idempotency key for driver app check-in/out (OWNER-APPROVED: Q3). [Source: report, §3.2, table `activity_sessions`; §3.7.1; OWNER-APPROVED: Q3.]
14. **`income_policies`** — One policy per driver/month; month is always day 1; base and trip rate are nonnegative. Store only formula inputs. Do not change a policy referenced by finalized payroll. [Source: report, §3.2, table `income_policies`; §3.7.2–§3.7.3.]
15. **`payroll_periods`** — One period per month; month is day 1. DRAFT has no finalizer/time; FINALIZED requires both. Finalize the whole period in one transaction after validating data and all payroll rows for drivers with activity in the period. [Source: report, §3.2, table `payroll_periods`; §3.7.3.]
16. **`payrolls`** — Unique per `(period_id, driver_id)`; policy must match both driver and period month. `trip_amount = completed_bookings × trip_rate`; counts and components are nonnegative; net follows the formula below. Do not finalize if `net_amount < 0`; resolve the data rather than clamp to zero. After finalization, payroll and source data affecting it cannot be changed. [Source: report, §3.2, table `payrolls`; §3.7.2–§3.7.3.]
17. **`payroll_adjustments`** — `amount > 0`; type determines whether it adds or subtracts. Payroll component totals equal the sums of detail rows by type and are recalculated transactionally before finalization. Do not apply the details a second time when printing. [Source: report, §3.2, table `payroll_adjustments`; §3.7.2–§3.7.3.]

### Cross-table integrity and deletion

Use `ON DELETE RESTRICT` (or equivalent) for historical business relationships; do not cascade-delete drivers, activity, leave requests, or payroll. Soft archive existing profiles and disable accounts. Archiving does not make a driver RESIGNED: resignation requires an effective RESIGNED employment interval. [Source: report, §3.6.1.]

Prevent time interval overlaps with exclusion constraints or transactionally locked triggers; UI-only checks are insufficient for concurrent writes. Check employment-history continuity separately because overlap checks do not detect gaps. `payrolls.policy_id` must match the driver's policy and the period month; a single-column FK alone does not guarantee this. [Source: report, §3.6.1–§3.6.2.]

## Timekeeping and payroll

For each closed session, duration is `(ended_at - started_at)` in minutes. Monthly completed trips are bookings assigned to the driver and completed in the local month of `completed_at`; sessions are not a trip source. Sum duration at original precision, then round the aggregate to two decimals. [Source: report, §3.7.1.]

Approved leave days count scheduled days covered by approved ANNUAL, SICK, LONG, or SUSPEND requests; post-resignation days are not leave. Unexcused absence counts scheduled days when the driver is WORKING, has no activity, and has no approved leave. A day with neither schedule nor activity is unregistered, not an unexcused absence. These metrics do not calculate deductions; every deduction needs a reasoned `payroll_adjustments` entry. Leave is date-based, not half-day/hour-based. [Source: report, §3.7.1.]

**Net formula:** `net = B + C*R + T + A - D`, where B is monthly base, C is completed assigned bookings, R is per-trip rate, and T/A/D are sums of BONUS/ALLOWANCE/DEDUCTION adjustments. Do not add detail rows twice in reports. Only COMPLETED bookings count. [Source: report, §3.7.2.]

### Six-step period lifecycle

1. Create a DRAFT period and include drivers with work time or business activity in the month, including drivers who left mid-month.
2. Verify each driver has a policy; no open sessions, overlapping/duplicate sessions, or leave/activity conflicts remain.
3. Aggregate activity and COMPLETED bookings; snapshot policy, trip count, driver code, and driver name into payroll.
4. Enter justified bonuses, allowances, and deductions; recalculate components and net. Draft recalculation replaces the row by `(period, driver)`, never duplicates it.
5. Lock the period and related data in one transaction, recheck conditions and totals, record finalizer/time, and set FINALIZED.
6. Prevent edits/deletes to finalized payroll, adjustments, policy, and source data affecting that period, including retroactive leave/status changes. Correct later errors with an adjustment in a later period; its description names the original period and reason. Never overwrite finalized history.

Drivers can view/print finalized payroll only; ADMIN may view clearly labelled drafts. Annual totals are grouped from finalized monthly periods; a month with no payroll displays “No payroll”, not zero income. [Source: report, §3.7.3.]

## Reports

- **Monthly driver status:** Separate end-of-month snapshot from changes during the month. For a closed month, use the state immediately before 00:00 on the next month’s first day. For the current month, use now and label it provisional through the stated date; do not use future scheduled states as actual. Include all drivers whose start date is no later than the report cutoff, including resigned drivers. The six employment statuses partition the total. Locked-account count is separate and may overlap any employment status. Report new hires by `start_date` in the month; distinguish resigned at month end from resignations starting during that month. Historical reports do not exclude currently archived profiles. [Source: report, §3.8.1.]
- **Workforce composition:** Age uses completed years at the report date. Group education by code; count distinct drivers by license class (one driver may appear in multiple classes). Vehicle group uses assignment effective at cutoff and includes “unassigned”. Area uses current area. Tenure runs from start date to report cutoff or earlier resignation date. Workdays/hours come from closed activity sessions; trip counts from COMPLETED bookings by `completed_at` month; income from FINALIZED payroll, averaged over drivers with a payroll row. For active drivers use tenure bands `<1 year`, `1–<3 years`, and `≥3 years`. Current education, area, and contact fields do not reconstruct historical values. Aggregate one-to-many license/assignment data separately to avoid multiplying income. [Source: report, §3.8.2.]
- **Payroll PDF:** Monthly output includes company name, period, driver code/name snapshot, workdays, hours, completed bookings, trip rate, base, adjustment details, net, and finalization time. Annual output has 12 month rows, payroll availability, and total of finalized months. Generate from finalized data; binary PDFs need not be stored in the database. [Source: report, §3.8.3.]

## Owner decisions

Each accepted default below is a decided, owner-approved rule as of 2026-10-27.

- **OWNER-APPROVED: Q1 (2026-10-27):** DRIVER may move their own booking to IN_PROGRESS or COMPLETED; ADMIN may transition any booking.
- **OWNER-APPROVED: Q2 (2026-10-27):** Cancellation is allowed from PENDING or ASSIGNED only.
- **OWNER-APPROVED: Q3 (2026-10-27):** Drivers start/end sessions in the app; client generates `source_key` as a UUID idempotency key.
- **OWNER-APPROVED: Q4 (2026-10-27):** ADMIN applies SUSPEND directly with `source_request_id = NULL`; keep SUSPEND request type for compatibility.
- **OWNER-APPROVED: Q5 (2026-10-27):** Enter `fare_amount` on completion; add a separate column if forecast fare is later required.
- **OWNER-APPROVED: Q6 (2026-10-27):** At effective resignation, reject login and refresh when RESIGNED is effective, and run a daily job to close `driver_vehicles` assignments.
- **OWNER-APPROVED: Q7 (2026-10-27):** For now, state the original period in adjustment `description`; add a source-period column if trace reporting is later needed.
- **OWNER-APPROVED: Q8 (2026-10-27):** Customer phone stays unique across the full table, including archived customers.
- **OWNER-APPROVED: Q9 (2026-10-27):** Redraw the BFD with three top-level branches: Management, Driver, Customer.
- **OWNER-APPROVED: Q10 (2026-10-27):** Use CHECK values for `education_level` as listed in the report; keep `license_class` free-form.

**No item in feedback §5 remains NEEDS-OWNER.** [Source: owner decision 2026-10-27; feedback §5.]
