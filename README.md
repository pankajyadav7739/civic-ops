# Civic Issues Reporting API

A backend API for citizens to report civic issues (potholes, drainage, waste management, etc.) with photos and audio notes, routed to relevant government departments. Built as a prototype during initial testing phases.

## Why This Exists

Managing civic complaints through fragmented channels (WhatsApp, calls, emails) means nothing gets tracked or prioritized. This API lets citizens report issues with location and media, admins see and route them to departments, and there's an audit trail.

## Features

**Phone-based authentication with OTP**  
Used Twilio for SMS rather than email because the target users are mostly on mobile with unreliable email access. OTPs expire in 5 minutes and include a 60-second retry cooldown to prevent brute force. JWT tokens with 24-hour expiry and 30-day refresh tokens.

**Issue reporting with file uploads**  
Citizens submit issues with title, description, category, location (GPS + address), and optionally an image or audio note. Files go to S3 instead of the database—keeps the DB lean and means users can upload without waiting for a server-side resize operation. Image and audio validation happens before upload (type and size checks). Issues get auto-assigned a custom ID in format `CIV{timestamp}`.

**Department routing**  
Categories map to departments via `CategoryDepartmentMapping`. When an issue is created, admins can assign it to the responsible department. This avoids hardcoding departments in code and makes it easy to reconfigure without redeployment. Department can be updated later if a mismatch occurs.

**Admin interface**  
Separate auth flow for admins (username/password + role-based permissions). Admins see all issues in their department, can mark status (reported → in_progress → resolved/rejected), and assign priority. Super admins can manage departments and see everything.

**Status tracking and audit**  
Issues track status changes and timestamps. Nothing gets deleted; everything is soft-deleted or marked with a timestamp so you can trace what happened.

## Stack

**FastAPI** — typed API framework with auto-generated docs. Faster iteration than Flask with better validation out of the box via Pydantic.

**PostgreSQL with SQLAlchemy** — relational database. SQLAlchemy ORM keeps queries readable and prevents SQL injection. Alembic migrations track schema changes and make rollbacks possible.

**S3 for file storage** — offloads media, avoids database bloat. boto3 client handles uploads. Files are public-read since issue photos should be visible to relevant departments.

**Twilio for SMS** — OTP delivery is reliable and cheaper than building SMS infrastructure. Fallback could be email but SMS is more reliable for this user base.

**JWT for auth** — stateless tokens so the API scales horizontally. Refresh tokens mean users don't need to re-verify OTP every day.

**Alembic for migrations** — tracks database schema evolution. One migration per change makes it easy to see what happened and when.

## How It Works

### Architecture

```
clients (web/mobile)
    ↓
FastAPI (routers)
    ↓
Services (auth, S3, OTP, admin)
    ↓
Database (PostgreSQL via SQLAlchemy)
    ↓
External (Twilio, S3)
```

**Request flow for issue creation:**

1. User authenticates via phone OTP
2. User calls `POST /users/issues/create` with form data (text fields + image file)
3. API validates file (size ≤ 10MB, type in whitelist)
4. File streams to S3, returns public URL
5. Issue record created in DB with S3 URL
6. Response includes custom ID and department assignment

**Request flow for admin:**

1. Admin logs in with username/password
2. Gets JWT token, refresh token
3. Calls `GET /admin/issues` to see their department's issues
4. Updates status via `PUT /admin/issues/{id}`

### Database Schema

**Users table**  
Stores citizen info. Phone number is unique (no duplicate registrations). UUID for external facing user ID. `is_profile_complete` flag unused currently but planned for profile enrichment.

**Issues table**  
The core data. `custom_id` is human-readable `CIV{timestamp}`. `photo_url` and `voice_note_url` are S3 URLs. `department_id` links to departments (nullable initially, set during admin review or via mapping). Priority starts as "unassigned" and admins can set it. Indexed on category and subcategory for fast filtering.

**OTPVerification table**  
Stores one-time codes with expiry. `retry_after` prevents request spam. Cleaned up automatically when old (could add a cron job to delete expired entries, not critical).

**AdminUser and Department tables**  
Admins have roles (admin = department-specific, super_admin = sees everything). Department has code (SWM, RMC, etc.) and name. Helps with reporting and filtering.

**CategoryDepartmentMapping table**  
Links categories + subcategories to departments. Allows the same category to be handled differently in different areas if needed. Indexed on category/subcategory for fast lookups.

### File Upload Strategy

Files stream directly to S3 instead of buffering on the server. Saves memory on the instance and works for large files. If S3 upload fails, the database transaction doesn't commit. This prevents orphaned S3 files (almost—race conditions are theoretically possible but rare).

File size limit is 10MB (configurable). Images are validated by MIME type before sending. Audio can be MP3, WAV, M4A, OGG.

### OTP and Auth

OTP is 6 digits, sent via Twilio SMS. Expires in 5 minutes. Two OTPs in 60 seconds are rejected (retry_after check). 

On first OTP verification, user is auto-created if they don't exist. On subsequent verifications, existing user is fetched. This means no explicit sign-up flow—the system treats OTP verification as account creation.

Access tokens last 24 hours. Refresh tokens last 30 days. Refresh token rotation isn't implemented (could add, but simpler without it for now).

### Department Routing

Issues don't auto-assign to departments. The idea was to make this configurable so admins could adjust routing rules later, but it ended up meaning admins manually assign departments per issue. Could be improved with a routing engine that checks the mapping table and auto-assigns.

### Why Certain Choices

**Alembic over Django migrations** — We use raw SQLAlchemy, not Django, so Alembic is the standard. Migrations are version-controlled SQL files.

**S3 over local storage** — Local storage doesn't scale across multiple instances. S3 is managed, versioned, and replicated.

**Public S3 URLs** — Issue photos should be visible to departments without needing presigned URLs for every request. Simpler but less private. For sensitive issues, could switch to presigned URLs (expires in 1 hour) but adds complexity.

**Phone-only auth** — Email unimplemented. Phone was priority. Email could be added, would need similar OTP flow or password reset.

**No password for citizens** — OTP is simpler UX for a mobile-first app. Passwords increase support burden (resets, forgotten passwords, etc.).

## Project Structure

```
app/
├── main.py              # FastAPI app setup, routes, health checks
├── models.py            # SQLAlchemy models (User, Issue, Department, etc.)
├── database.py          # SQLAlchemy engine and session setup
├── config.py            # Environment variables (Twilio, S3, JWT secrets)
├── requirements.txt     
├── core/
│   ├── security.py      # JWT creation, OTP generation, password hashing
│   ├── dependencies.py  # FastAPI dependency injection (current user, etc.)
│   └── admin_dependencies.py  # Admin auth dependencies
├── routers/
│   ├── auth.py          
│   ├── users.py         # Issue creation, user profile
│   └── admin.py         # Department and issue management
├── services/
│   ├── auth_service.py  # OTP flow, token generation
│   ├── otp_service.py   # Twilio integration
│   ├── s3_service.py    # S3 upload, validation
│   └── admin_auth_service.py  # Admin login, permissions
├── schemas/
│   ├── auth.py          # OTP request/response schema
│   ├── issues.py        # Issue creation/response schema
│   └── admin.py         # Admin login schema
└── utils/
    └── file_utils.py    # File validation helpers

alembic/
├── env.py               # Alembic configuration
├── script.py.mako       # Migration template
└── versions/            # Versioned migrations (one per DB change)
```

## Running Locally

**Prerequisites:**
- Python 3.8+
- PostgreSQL
- AWS S3 bucket
- Twilio account

**Setup:**

```bash
# Clone and navigate to project
git clone <repo>
cd civic-ops

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # or `venv\Scripts\activate` on Windows

# Install dependencies
pip install -r app/requirements.txt

# Create .env file in project root
cat > .env <<EOF
DATABASE_URL=postgresql://user:password@localhost:5432/civic_ops
JWT_SECRET=your-secret-key-change-this
TWILIO_ACCOUNT_SID=your-twilio-sid
TWILIO_AUTH_TOKEN=your-twilio-token
TWILIO_PHONE_NUMBER=+1234567890
AWS_ACCESS_KEY_ID=your-aws-key
AWS_SECRET_ACCESS_KEY=your-aws-secret
AWS_REGION=us-east-1
S3_BUCKET_NAME=your-bucket
EOF

# Run migrations
alembic upgrade head

# Start the server
cd app
uvicorn main:app --reload

# API docs at http://localhost:8000/docs
```

**Running on Render (deployment):**

The `start.sh` script uses Gunicorn with Uvicorn workers. Render runs this automatically on deployment.

## Testing

A few test files exist in the root:

- `test_s3_upload.py` — verifies S3 connectivity and file upload
- `test_issue_detail.py` — tests issue creation and retrieval
- `test_enhanced_issues.py` — tests multipart form submission
- `test_s3_config.py` — checks S3 bucket configuration

Run with `python test_name.py` (make sure `.env` is configured and server is running).

Setup scripts:

- `setup_admin_users.py` — Create admin accounts
- `setup_category_mappings.py` — Populate category-to-department mappings
- `configure_s3.py` — Test S3 configuration

## Notes / Limitations

**Department routing is manual** — Issues don't auto-assign to departments. An admin has to review and assign. Could be improved with rule-based routing, but for a pilot this is fine.

**No soft deletes** — Issues and users are never deleted, but there's no explicit soft-delete column. Could add an `is_deleted` flag if needed for business logic (e.g., hiding resolved issues by default).

**File cleanup** — If S3 upload succeeds but database commit fails, the S3 file is orphaned. Could mitigate with periodic cleanup of files not referenced in the database.

**OTP retry logic** — The 60-second retry is global per phone number. If someone legitimately needs a second OTP (first didn't arrive), they have to wait. Twilio has built-in retry logic so this is usually fine, but could be more lenient.

**No rate limiting** — The API accepts unlimited requests from a single IP. In production, should add rate limiting per user and globally to prevent abuse.

**Admin roles not fully enforced** — The schema has `role` (admin vs super_admin) but endpoints don't fully check it. Super admins *should* see all departments but currently anyone can see anything if they're logged in as admin.

**Audio/image processing** — Files are uploaded as-is. No resizing, compression, or format conversion. Large videos would fail the size check. Could add async processing (upload to S3, trigger Lambda for processing) if needed.

**No error recovery UI** — If an issue creation fails halfway (e.g., S3 timeout), the user sees the error but the DB might have a partial record. Transactions help but race conditions are possible.

**Email auth unimplemented** — Only phone OTP works. Email auth would be useful for desktop users but wasn't prioritized.

## Future Work

**Automatic department routing** — Run a query against `CategoryDepartmentMapping` on issue creation and auto-assign. Remove manual assignment.

**Issue priority auto-calculation** — Machine learning to predict priority based on category, location history, time of day, etc. Start simple (category-based weights).

**Admin mobile app** — Push notifications when new issues arrive. Web dashboard works but mobile is better for field officers.

**Escalation workflow** — If an issue isn't resolved in 30 days, auto-escalate to supervisor. Simple timer in background task.

**Analytics dashboard** — Charts of issues by category, resolution time, department performance.

**Bulk upload** — CSV import for historical data (legacy systems).

**API rate limiting** — Per-user and global rate limits to prevent abuse.

**Twilio integration improvements** — Support WhatsApp and SMS replies as another way to update issues.

## Contributing

This is a pilot project. If you're adding features, keep the following in mind:

- Write migrations for every schema change. Never alter tables manually.
- Add Pydantic schemas for every endpoint request/response. Validation is free.
- Keep business logic in services, not routers. Routers should only handle HTTP concerns.
- Test S3 upload before deploying (it's easy to misconfigure).
- Document env variables in `.env.example` if adding new ones.


**Deployed at:** https://civic-ops.onrender.com  
**Status:** Pilot phase (small user base, expect bugs)  
**Last updated:** May 2026
