# Concurrency-Safe Seat Reservation and Consistency Model

## Reservation Model
- Each seat row stores `locked_by` and `lock_expires_at`.
- A reservation lasts 2 minutes (`SEAT_LOCK_TIMEOUT`).
- Locks are created only inside a database transaction.

## How Double Booking Is Prevented
1. User selects one or more seats.
2. Server enters `transaction.atomic()` in `movies/seat_locking.py`.
3. Selected seat rows are fetched with `select_for_update()`.
4. Django/PostgreSQL/SQLite writer lock ensures concurrent writers cannot both mutate the same seat rows at the same time.
5. Server checks three conditions before writing locks:
   - seat exists in the target show (`theater` record in current model)
   - seat is not already booked
   - seat is not actively locked by another user with an unexpired lock
6. Only if all checks pass, server writes `locked_by=user` and `lock_expires_at=now + 2 minutes`.

This gives single-writer safety for the same seat rows and prevents race-condition double booking even if two users submit within milliseconds.

## Auto Timeout and Background Release
- Expired locks are released by `release_expired_seat_locks()`.
- Background scheduler command:
  - `python manage.py release_expired_reservations --interval 15`
- This command continuously clears expired reservations without requiring page refresh.

## Edge Cases Covered
- User closes app/browser:
  - lock naturally expires and scheduler clears it.
- Network interruption after locking seats:
  - seat remains reserved only until `lock_expires_at`, then becomes available again.
- User opens multiple devices/tabs:
  - lock ownership is checked against the authenticated user and current expiry timestamp.
- Payment page left open after timeout:
  - polling endpoint `/movies/payment/lock-status/` detects invalid lock and forces user back to seat selection.

## Consistency Model
- Seat locking and booking finalization are strongly consistent at seat-row level.
- The system uses transactional consistency with row-level locking (`select_for_update`) to serialize conflicting seat updates.
- Asynchronous flows such as payment callback vs webhook are eventually consistent, resolved by idempotent payment finalization and duplicate-event suppression.

## Demonstration Checklist
- Two users pick the same seat at nearly the same time: one succeeds, the other receives a seat-locked error.
- User leaves locked seats idle for more than 2 minutes: scheduler releases them automatically.
- Same user opens checkout on another device after lock expiry: payment is rejected because lock is no longer valid.
