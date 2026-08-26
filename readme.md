Here's a more descriptive version of both:

---

## Ticket comment (expanded)

```
 Please find the below analysis summary and the resolution

The 400 is expected behavior — this is a legitimate validation catch, not a code bug, and I've confirmed there is no data corruption from the failed request.

Trace Details
Trace ID: 275544847252093555 | Venue: 10719 (Marea) | Timestamp: Aug 24, 2026 1:21:14 PM MST | Service: inventory (v192.7) | Duration: 306ms | Caller: os-web → inventory

Root Cause
The Schedule Manager correctly detected a genuine overlap between two schedules:
- Schedule 2953964: 2024-03-25 → 2024-04-09
- Schedule 2954062: 2024-04-09 → 2024-04-09

Both ranges include April 9 as a boundary date, so the overlap check treats them as inclusive on both ends. This triggered _assert_no_overlaps() at Manager.py:474, which raised ScheduleOverlapError, caught at Shift.py:133 and converted into the HTTP 400 the partner saw.

Deployment check: No production deployment occurred before this error. Production inventory was stable on v192.7 (commit 288e5b5a) throughout. The only deploys around that time were to dev/test environments (e1/e2), unrelated to this request. This rules out a regression from a recent release.

Data Integrity — Confirmed Safe
Since the Schedule Manager performs several DB writes (inserts/deletes) before running the overlap check, I verified whether those writes persist despite the 400. They do not.

The app follows a commit-only-on-success pattern: self.commit() (line 184) only executes on the happy path. Any exception — including ScheduleOverlapError — means commit() is never reached. The teardown hook (_resy_shutdown()) is registered in two places (the error handler and the after-request hook) and always calls rollback(True) on error, which reverts all uncommitted work:
- shift.upsert() — rolled back
- INSERTs into shift_config_algorithm, shift_config_availability — rolled back
- INSERTs into schedule_info, schedule_shift — rolled back
- DELETEs from schedule_shift, shift_floorplan — rolled back

The database is returned fully to its pre-request state. The 400 response is purely informational — no partial or orphaned rows exist for venue 10719 from this request. There's also a row-level lock (SELECT ... FOR UPDATE) acquired at the start of execute() that prevents concurrent schedule edits on the same venue/date-range from racing each other during the ~306ms request window; this lock releases on rollback too, so it doesn't cause any lingering lock contention.

Resolution
No code fix needed — this is working as intended. The venue was attempting to save two shift configurations that both claim April 9 as part of their range, which is a genuine scheduling conflict, not a system error. Recommend advising the venue to adjust one of the two ranges so they don't share a boundary date before re-saving.

Closing as Cannot Reproduce / Working as Expected — no data integrity risk identified.

CC:
```

---

## Slack reply (expanded, still scannable)

```
Confirmed — ran the full trace + code investigation on this one.

The 400 is expected, not a bug. The Schedule Manager caught a real conflict: two shift schedules for Marea both include April 9 as a boundary date (one runs 3/25→4/9, the other 4/9→4/9), so the overlap validator correctly rejected the save.

Also ruled out deployment as a cause — production was stable on the same version (v192.7) the whole time, no releases landed before this happened.

The part I really wanted to nail down: whether the DB writes that happen *before* the overlap check get left behind when it fails. They don't — traced through the transaction handling and confirmed the app only commits on success; any error triggers a full rollback, so the database goes right back to its pre-request state. No orphaned data, no risk from repeated failed attempts.

So: working as intended, no fix needed on our end. Just a genuine overlapping-date conflict the venue needs to adjust. Posting the full writeup on the ticket now.
```
