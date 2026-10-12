# Employee Attendance & Analytics API

Submission for Software Engineer Trainee role at HROne.

## Tech Stack
- **Framework**: FastAPI (Python 3.11+)
- **Server**: Uvicorn
- **Database**: MongoDB 6.0+ (Tested against MongoDB 7.0/8.x)
- **ODM/Driver**: PyMongo
- **Validation**: Pydantic v2

## How to Run

1. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

2. **Configure environment (Optional):**
   Environment variables `MONGO_URI` and `MONGO_DB` are read dynamically with fallback to local defaults:
   - `MONGO_URI` (defaults to `mongodb://localhost:27017`)
   - `MONGO_DB` (defaults to `attendance_db`)

3. **Seed sample data (Optional):**
   ```bash
   python sample_seed.py
   ```

4. **Start the API:**
   ```bash
   uvicorn app.main:app --port 8000
   ```
   Interactive API docs are available at `http://localhost:8000/docs`.

## Project Structure
- `app/main.py`: Complete FastAPI application containing all endpoints, business rules (R1–R10), aggregation pipelines, and startup index creation.
- `requirements.txt`: Python dependencies.
- `REVIEW.md`: Comprehensive code review identifying and fixing 16 defects in the starter code.
- `DECISIONS.md`: Design rationale covering indexes, concurrency race handling, tie ranking, headcount calculation, and 100x scaling.
- `openapi.yaml`: API specification and contract.
- `DATA_MODEL.md`: MongoDB schema documentation.

## Features & Implementation Highlights
- **Startup Index Creation**: Lifespan handler idempotently creates all indexes on `employees` and `attendance_logs` for `IXSCAN` execution.
- **Race-Safe Attendance**: Atomic database operations and compound unique indexes prevent duplicate punch-ins and double punch-outs under concurrency.
- **Overnight Shift Handling**: Accurately maps punch-ins before `shift_end` to the shift starting on the previous calendar day (Rule R1).
- **Rounding & Precision**: Strict half-up rounding (R8) using `decimal.Decimal(..., rounding=ROUND_HALF_UP)`.
- **Pure Database Aggregations**:
  - `employee_monthly`: Monthly summaries with working days computed from `joined_on`.
  - `department_summary`: Cross-collection `$lookup` ensuring employees with zero logs are counted in headcount (R9).
  - `late_leaderboard`: Standard competition ranking (1, 2, 2, 4) using MongoDB `$setWindowFields` with `$rank`.
  - `department_trend`: Calendar gap-filling via `$densify` and 7-day moving averages via `$setWindowFields`.
  - `/admin/explain/{endpoint}`: Returns raw MongoDB `executionStats` confirming index scans (`IXSCAN`) and zero collection scans (`COLLSCAN`).
