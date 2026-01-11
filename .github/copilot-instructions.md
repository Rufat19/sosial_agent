# Copilot Instructions: DSMF Citizen Request Bot

## Project Overview

**DSMF** is a Telegram bot that collects citizen requests through a multi-step form, routes them to an executor group with inline action buttons, and provides real-time status updates via direct messages.

**Core Tech Stack:**
- `python-telegram-bot` 21.6 (PTB) with async polling
- PostgreSQL (primary) + SQLite (automatic fallback)
- SQLAlchemy ORM with context managers for session safety
- Bakı timezone-aware timestamps (UTC+4)

**Entry Points:**
- `run.py` - Boot script (sets UTF-8 encoding, adds `src/` to path, calls `bot.py:main()`)
- `src/bot.py` - Bot logic, conversation states, handlers, error handling (1579 lines)
- `src/config.py` - Environment variables, validation, logging, message templates

---

## Architecture & Data Flow

### 1. **Conversation Pipeline (User ↔ Bot)**

The form is implemented as a `ConversationHandler` with sequential states (enum in `bot.py: States`):
1. `FULLNAME` - Name validation (min 2 chars, non-empty)
2. `PHONE` - Phone number parsing via `phonenumberslite`
3. `ID_TYPE` - Choice: Passport (FIN) vs. Residency (PIN)
4. `FIN` or `PIN` - 7-char FIN (letters+digits) or 5-6-char PIN
5. `ID_PHOTO` - Telegram file_id (stored for audit)
6. `FORM_TYPE` - Dropdown: Complaint (Şikayət), Suggestion (Təklif), Application (Ərizə)
7. `BODY` - Free-text (10-350 chars) → saved to DB
8. `CONFIRM` - Review & submit

**Key Detail:** Subject is auto-generated from first 50 chars of body; no separate subject field.

### 2. **Executor Group Workflow (Inline Actions)**

When submitted, bot sends formatted message to `EXECUTOR_CHAT_ID`:
- Shows: ID, Citizen name, Phone, FIN/PIN, Form type, Body
- Inline buttons: `✉️ Respond` (callback: `reply_*`) and `🚫 Reject` (callback: `reject_*`)
- Status line updates: 🟡 Waiting | 🔴 Overdue | 🟢 Answered | ⚫ Rejected

Executor clicks button → `CallbackQueryHandler` triggers → prompts for text/reason → DB update → message edit in group (status refreshes) → DM to citizen.

### 3. **Database Abstraction Layer**

**Primary: PostgreSQL**
- `src/db_operations.py` - All DB calls wrapped in `with get_db()` context manager
- `src/database.py` - SQLAlchemy ORM models: `Application`, `BlacklistedUser`
- Enums: `ApplicationStatus` (waiting|answered|rejected), `FormTypeDB` (complaint|suggestion|application)
- **Critical:** Always `db.expunge(app)` before returning to avoid "Instance not bound to Session" errors

**Fallback: SQLite**
- `src/db_sqlite.py` - Direct SQL (no ORM) for `data/applications.db`
- Auto-activated if PostgreSQL fails at startup or `FORCE_SQLITE=1` env var
- `/export` returns JSON instead of CSV

**Activation Logic** (bot.py ~line 80):
```python
try PostgreSQL (db_operations)
except ImportError → try SQLite (db_sqlite)
except → DB_ENABLED = False (no DB)
```

---

## Developer Workflows

### Local Testing (SQLite, No PostgreSQL Required)
```powershell
$env:FORCE_SQLITE="1"
.\.venv\Scripts\python.exe run.py
```
Creates `data/applications.db`, `/export` returns JSON.

### Production (Railway)
```bash
git push origin main
```
- Railway detects `Procfile`, `runtime.txt`, `railway.json`
- Auto-installs `requirements.txt` (includes psycopg2-binary)
- Sets `DATABASE_URL` when PostgreSQL service attached
- Bot polls with `drop_pending_updates=True` + 30s timeouts

### Polling Conflict Fix (Railway)
Error: "Conflict: terminated by other getUpdates request"
1. Settings → Scaling → verify `replicas = 1`
2. Rotate bot token via @BotFather
3. Stop old deployments, keep only latest

---

## Critical Patterns & Conventions

### 1. Configuration & Secrets
- **Source:** `.env` (never committed; use `.env.example`)
- **Validation:** `config.py` raises `ValueError` if `BOT_TOKEN` missing
- **Admin Control:** `ADMIN_USER_IDS` (comma-separated → set of ints)
- **Access:** Only admins can run `/ban`, `/unban`, `/clearall`, `/export`

### 2. Form Type Mapping
| Display | Callback | DB Enum | Python |
|---------|----------|---------|--------|
| Şikayət | complaint_btn | complaint | FormType.COMPLAINT |
| Təklif | suggestion_btn | suggestion | FormType.SUGGESTION |
| Ərizə | application_btn | application | FormTypeDB.APPLICATION |

**When saving:** Convert user choice → `FormTypeDB` enum → ORM, then string → SQLite.

### 3. Status System
- `ApplicationStatus.PENDING` (value: "waiting") → 🟡 **Waiting**
- `ApplicationStatus.COMPLETED` (value: "answered") → 🟢 **Answered**
- `ApplicationStatus.REJECTED` (value: "rejected") → ⚫ **Rejected**
- **Overdue:** 🔴 if `created_at <= now() - 10 days`

Update pattern:
```python
from db_operations import update_application_status
from database import ApplicationStatus

update_application_status(app_id=123, 
    status=ApplicationStatus.COMPLETED,
    reply_text="Response text")
```

### 4. Timezone & Timestamps
- **Always:** Use `BAKU_TZ` (defined in `config.py`) for all dates
- **Example:** `datetime.now(BAKU_TZ)` for form submission
- **CSV Export:** Converts all times to `DD.MM.YYYY HH:MM:SS` (Bakı TZ)
- **Why:** Citizens see consistent local time, no UTC confusion

### 5. Message Templates
All user-facing text in `config.py: MESSAGES` dict
- Single source of truth; no hardcoded strings in `bot.py`
- Keys: "welcome", "phone_error", "phone_prompt", etc.
- Language: `LANG=az` env var (Azerbaijani only currently)

### 6. Error Handling
- **Telegram API:** Try/except with context logging (user ID, chat ID)
- **Polling Conflict:** Special case in `error_handler()` → logs warning, does NOT raise
- **Database:** Session context auto-rolls back on exception
- **Input Validation:** Return user-friendly errors, never raise exceptions

### 7. Session & State Management
- **User Data:** Access via `_ud(context)` helper → always returns mutable dict
- **Conversation State:** Stored in `context.user_data` during form
- **Cleanup:** Auto-cleared when form completes/cancels
- **Type Safety:** Use `ApplicationData` dataclass (bot.py ~line 140) for form data

### 8. Rate Limiting & Blacklist
- **Daily limit:** `MAX_DAILY_SUBMISSIONS = 3` (24-hour rolling window)
- **Auto-blacklist:** ≥5 rejections in 30 days → `blacklisted_users` table → `/start` blocked
- **Admin bypass:** Users in `ADMIN_USER_IDS` exempt from both

Check pattern:
```python
if is_user_blacklisted(user_id):
    return "❌ Access denied"
if count_user_recent_applications(user_id, hours=24) >= MAX_DAILY_SUBMISSIONS:
    return "⚠️ Limit exceeded"
```

---

## File Map

| File | Purpose |
|------|---------|
| `run.py` | Boot: UTF-8 setup, path config, calls `bot.main()` |
| `src/bot.py` | 1579-line core: conversation, handlers, polling |
| `src/config.py` | Env vars, validation, logging, messages, timezone |
| `src/database.py` | SQLAlchemy models & enums (PostgreSQL) |
| `src/db_operations.py` | ORM API with session context manager |
| `src/db_sqlite.py` | Direct SQL fallback (no ORM) |
| `.env` | Secrets: BOT_TOKEN, EXECUTOR_CHAT_ID, admin IDs |
| `README.md`, `COMMANDS.md`, `DATABASE.md` | User-facing docs |
| `Procfile`, `runtime.txt`, `railway.json` | Railway deployment |

---

## Integration Points

### Telegram API
- **Polling:** `Application.run_polling()` with `drop_pending_updates=True`
- **File Storage:** ID photos stored as Telegram `file_id` (not downloaded; cost-effective)
- **Inline Buttons:** `CallbackQueryHandler` manages executor group actions

### PostgreSQL + Railway
- **Connection:** Via `DATABASE_URL` env var (Railway provides automatically)
- **Public Proxy:** Use Railway's public hostname, not internal (external apps require this)
- **Fallback:** If PostgreSQL unavailable at startup, SQLite auto-activates

### Admin Commands
- `/blacklist` - List all blacklisted users
- `/ban <user_id> [reason]` - Add to blacklist
- `/unban <user_id>` - Remove from blacklist
- `/clearall` - Delete all applications (test only; ID sequence resets)
- `/export` - CSV (PostgreSQL) or JSON (SQLite)

---

## Quick Reference: Common Tasks

### Add New Admin
1. Get user ID: send `/chatid` in private chat
2. Edit `.env`: `ADMIN_USER_IDS=123,456,<new_id>`
3. Restart bot

### Change Message Text
Edit `config.py: MESSAGES` dict:
```python
MESSAGES["phone_prompt"] = "📱 Enter phone..."
```

### Extend Form (Add Field)
1. Add state to `States` enum in `bot.py`
2. Add handler `async def handle_new_field()`
3. Update `ConversationHandler` route
4. Add DB column (alter table in `database.py` / `db_sqlite.py`)

### Debug DB Issues
Search for `db_operations.*()` calls, verify session context manager usage, check `db.expunge()` calls.

### Fix "Instance not bound to Session"
Always call `db.expunge(obj)` before returning from `db_operations` functions.

---

## Common Pitfalls

1. **Enum Confusion:** `FormType` (Python, user-facing) ≠ `FormTypeDB` (SQLAlchemy). Convert carefully.
2. **Timezone Bugs:** Never use `datetime.now()`. Always: `datetime.now(BAKU_TZ)`.
3. **Session Lifecycle:** Context manager auto-commits; don't call outside.
4. **Missing Admin Check:** `/ban`, `/export` must verify `update.effective_user.id in ADMIN_USER_IDS`.
5. **Polling Conflicts:** Railway needs `replicas = 1`; old deployments must stop.

---

## References

- **Internal:** README.md (features, deploy), COMMANDS.md (all commands), DATABASE.md (schema)
- **External:** [python-telegram-bot](https://python-telegram-bot.readthedocs.io/), [SQLAlchemy ORM](https://docs.sqlalchemy.org/)
- **Deployment:** DEPLOYMENT.md (Railway step-by-step)
- **Telegram:** @BotFather (token & testing)
