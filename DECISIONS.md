# DECISIONS.md

1. **Indexes.**
   - On `employees`: `{ emp_code: 1 }` (unique) ensures business-key uniqueness and fast point lookups; `{ department: 1, joined_on: 1 }` accelerates department-filtered headcount and pagination.
   - On `attendance_logs`: `{ emp_code: 1, date: 1 }` (unique) guarantees at most one record per employee/day and prevents punch-in race conditions; `{ date: -1, emp_code: 1 }` and `{ emp_code: 1, date: -1 }` support fast range filtering and ordered pagination without memory sorts; `{ emp_code: 1, punch_in: -1 }` enables sub-millisecond open punch-in lookups; `{ date: 1, late_minutes: 1 }` allows index scans for the late leaderboard.
   - *Rejected:* An index on `status` alone was rejected because low cardinality yields poor selectivity and unnecessary write overhead.

2. **Punch-in race.**
   - Two concurrent punch-in requests calculate the attendance date and attempt `insert_one`.
   - WiredTiger serializes document insertion via the unique index on `(emp_code, date)`. The first request commits and returns `201 Created`.
   - The second request trips MongoDB's unique index constraint, raising `DuplicateKeyError`, which our application catches and maps directly to `409 Conflict`. There is no check-then-insert race window.

3. **Ties.**
   - The leaderboard uses `$setWindowFields` with `$rank`, implementing standard competition ranking (1, 2, 2, 4).
   - If two employees tie at `rank <= limit`, both are returned (potentially returning more than `limit` rows). Tied rows are ordered by `total_late_minutes` descending, then deterministically by `emp_code` ascending.

4. **Headcount.**
   - The aggregation pipeline begins on `employees` filtering `joined_on <= end_of_month` (not on logs).
   - It performs a `$lookup` on `attendance_logs` and unwinds with `preserveNullAndEmptyArrays: true`. Department headcount is derived from distinct employee codes, guaranteeing employees with zero logs are accurately counted.

5. **One thing you would change for 100x data.**
   - At 100x data (~10 million records), we would shard `attendance_logs` using a compound shard key `(emp_code, date)`. To keep analytics instantaneous, we would maintain daily/monthly rollups via change streams or background workers rather than running cross-collection `$lookup` joins on raw logs on the fly.
