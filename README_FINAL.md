# Real-Time Seat Allocation Engine (Flash-Sale Ticket Booking System)

## 1. Problem Statement

When ticket sales open for a popular event, a limited number of seats can be hit by thousands of
users trying to book at the exact same moment. A naive booking system either:
- Lets two users book the same seat (a double-booking bug), or
- Makes every user queue up behind whoever locked a seat row first, causing massive slowdowns
  exactly when the system needs to be fastest.

This project solves both problems using Oracle's `SKIP LOCKED` row-locking feature, combined with
a two-phase booking flow (seat is *held* first, then *confirmed* once payment succeeds) and an
automated cleanup process for holds that are never paid for.

## 2. Database Design

| Table | Purpose |
|---|---|
| `events` | The show/match being sold (name, venue, date, total seats) |
| `seats` | Every individual seat: class (VIP/PREMIUM/GENERAL), status (available/held/booked), price |
| `users` | People who can book seats |
| `bookings` | Each booking attempt: which user, which seat, and its status (held/confirmed/cancelled/expired) |
| `payments` | Payment attempts tied to a booking (success/pending/failed) |
| `audit_log` | A full history trail — every action (held/confirmed/cancelled/expired) is logged automatically |

## 3. The Core Engine — `pkg_seat_allocation` Package

All booking logic lives in one PL/SQL package, containing:

- **`proc_hold_seat`** — finds and locks one available seat using
  `SELECT ... FOR UPDATE SKIP LOCKED`, marks it held, creates a booking record, and logs the action —
  all as one atomic transaction.
- **`proc_confirm_booking`** — marks a held booking as confirmed once payment succeeds, and updates
  the seat to booked.
- **`proc_cancel_booking`** — cancels a booking and immediately frees the seat back to available.
- **`proc_release_expired_holds`** — finds every booking held for more than 5 minutes without
  payment, and releases those seats in bulk using `BULK COLLECT` and `FORALL` (efficient even with
  large numbers of expired holds at once).
- **`proc_log_action`** — a shared helper that every other procedure calls to automatically write
  an audit trail entry, so logging can never be forgotten or skipped.

## 4. Why `SKIP LOCKED` Matters (The Key Concept)

A normal lock (`SELECT ... FOR UPDATE`) makes a second user *wait* if they try to grab a row someone
else is already handling. Under a flash-sale rush, this creates a queue and slows the whole system
down at the worst possible moment.

`SKIP LOCKED` changes the approach: instead of asking for one *specific* seat, the procedure asks for
*any* available seat in the requested class. If a candidate seat is already locked by another
session, it's skipped instantly, and the next available one is tried instead. Nobody waits, and two
users can never walk away with the same seat.

## 5. Concurrency Proof — The Centerpiece Demonstration

To prove this works, two separate SQL sessions were run at the same time, both attempting to book
the last remaining VIP seat via `proc_hold_seat`.

**Result:**
- One session received a valid booking ID immediately.
- The other session received an instant "Sold out" message — with no waiting, no error, and no
  double-booking.

This is direct evidence that the locking strategy correctly handles simultaneous requests under
real concurrent load, which is the entire problem this project was built to solve.

*(See attached screenshot: `concurrency_test_result.png`)*

## 6. Full Lifecycle Tested

Every stage of a booking's life was individually tested against the running package:

| Action | Procedure | Result Verified |
|---|---|---|
| Hold a seat | `proc_hold_seat` | Seat → held, booking created, audit logged |
| Confirm a booking | `proc_confirm_booking` | Booking → confirmed, seat → booked, confirm_time recorded |
| Cancel a booking | `proc_cancel_booking` | Booking → cancelled, seat → available again |
| Expire an unpaid hold | `proc_release_expired_holds` | Booking → expired, seat → available, using BULK COLLECT/FORALL |

## 7. Challenges Solved During Development

- **CHECK constraint mismatches** — status values used by the PL/SQL code (e.g. `'cancelled'`) had
  to exactly match what the table constraints allowed, down to spelling (`'cancelled'` vs
  `'canceled'`) and case sensitivity.
- **Identity column behavior** — learned that `DELETE` does not reset an identity counter, while
  `TRUNCATE` does, and how to manually restart one when needed.
- **Data integrity across related tables** — experienced firsthand what happens when `bookings` and
  `seats` fall out of sync, which reinforced exactly why wrapping related updates inside a single
  PL/SQL procedure (rather than doing them manually) is necessary.
- **RETURNING clause syntax** — learned that `RETURNING ... INTO ...` must be part of the same
  `INSERT` statement, not a separate statement afterward.

## 8. What I'd Add With More Time

- Full `DBMS_SCHEDULER` automation to run `proc_release_expired_holds` every minute automatically,
  rather than calling it manually.
- A `payments` integration so `proc_confirm_booking` is only callable after a real successful
  payment record exists.
- A waitlist/RAC-style queue for sold-out seat classes, instead of a flat "sold out" response.
- A simple front-end (even a basic form) to call these procedures instead of running them from
  SQL*Plus directly.

## 9. Summary

This project demonstrates a working, tested, real-time seat booking engine that correctly handles
concurrent access using Oracle's `SKIP LOCKED` feature, manages a full booking lifecycle
(held → confirmed / cancelled / expired) through a single PL/SQL package, and maintains a complete
audit trail of every action — built and debugged end-to-end against a live Oracle database.
