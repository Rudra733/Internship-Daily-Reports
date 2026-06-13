# Trial Provisioning — Data Models & API

These extend your existing `demo_requests` table. Nothing here replaces what you
already capture (school profile, operations, requested features) — it adds the
**roles → seats → accounts → expiry** layer.

## Tables

### 1. `demo_requests` (existing — add a few columns)
```sql
ALTER TABLE demo_requests
  ADD COLUMN trial_status   ENUM('not_started','active','expired','converted','lost')
                            DEFAULT 'not_started',
  ADD COLUMN trial_start    DATE NULL,
  ADD COLUMN trial_end      DATETIME NULL,   -- single source of truth for expiry
  ADD COLUMN trial_days     SMALLINT DEFAULT 7;
```
Keeping the trial window on the request (not per account) means every role expires
together — simpler and what schools expect. Per-account expiry is only needed if you
provision seats at different times.

### 2. `demo_request_roles` — what the school asked for vs. what you grant
```sql
CREATE TABLE demo_request_roles (
  id              BIGINT PRIMARY KEY AUTO_INCREMENT,
  demo_request_id BIGINT NOT NULL REFERENCES demo_requests(id),
  role            ENUM('principal','coordinator','teacher',
                       'accountant','librarian','hostel_warden',
                       'transport_manager','front_desk') NOT NULL,
  requested_count INT NOT NULL DEFAULT 0,   -- demand signal for sales
  trial_seats     INT NOT NULL DEFAULT 0,   -- what you actually provision (<= cap)
  UNIQUE (demo_request_id, role)
);
```
`requested_count` is the conversion signal — keep it even when `trial_seats` is capped
to 1. The default trial cap lives in config, not hardcoded.

### 3. `trial_accounts` — the actual login credentials
```sql
CREATE TABLE trial_accounts (
  id              BIGINT PRIMARY KEY AUTO_INCREMENT,
  demo_request_id BIGINT NOT NULL REFERENCES demo_requests(id),
  role            VARCHAR(32) NOT NULL,
  full_name       VARCHAR(120),
  email           VARCHAR(190) NOT NULL UNIQUE,
  status          ENUM('active','expired','terminated') DEFAULT 'active',
  password_hash   VARCHAR(255) NOT NULL,    -- bcrypt/argon2 of the admin-set password
  must_reset      BOOLEAN DEFAULT TRUE,     -- force change on first login (recommended)
  activated_at    DATETIME NULL,            -- first successful login
  expires_at      DATETIME NOT NULL,        -- copied from trial_end at creation
  terminated_at   DATETIME NULL
);
```
SproutSong admin sets the email and password for each seat (that's what you wanted),
so the account is usable immediately. Store only the **hash**, never the plaintext — and
keep `must_reset = TRUE` so the school user is prompted to set their own password on first
login. That keeps your manual-creation flow while avoiding the risk of a shared plaintext
password living in the database. The admin UI can still display the password once at
creation time so it can be handed over.

### 4. `demo_request_features` — what was ticked in Step 4
```sql
CREATE TABLE demo_request_features (
  id              BIGINT PRIMARY KEY AUTO_INCREMENT,
  demo_request_id BIGINT NOT NULL REFERENCES demo_requests(id),
  module_group    ENUM('academic','staff','engagement','erp','ai','analytics') NOT NULL,
  feature_key     VARCHAR(64) NOT NULL,    -- e.g. 'exam_result_automation'
  UNIQUE (demo_request_id, feature_key)
);
```
This is the raw output of your 5-step form's Step 4. Each feature rolls up to one of the
six module groups, so you can answer "what did they ask for" at either level.

### 5. `trial_modules` — which groups you switch on for the trial
```sql
CREATE TABLE trial_modules (
  id              BIGINT PRIMARY KEY AUTO_INCREMENT,
  demo_request_id BIGINT NOT NULL REFERENCES demo_requests(id),
  module_group    ENUM('academic','staff','engagement','erp','ai','analytics') NOT NULL,
  enabled         BOOLEAN DEFAULT FALSE,
  UNIQUE (demo_request_id, module_group)
);
```
A school that wants **ERP only**, or only **AI Tools**, or only **Analytics** is just a
row pattern here — the trial enables those groups and leaves the rest locked. This table is
the seed for the future SaaS-module gating: when you go paid, the same group keys drive
per-plan access, and a middleware check on each module reads `enabled`.

### 6. `trial_events` — audit log
```sql
CREATE TABLE trial_events (
  id              BIGINT PRIMARY KEY AUTO_INCREMENT,
  demo_request_id BIGINT NOT NULL,
  account_id      BIGINT NULL,
  event           VARCHAR(40) NOT NULL,  -- provisioned, invited, activated,
                                         -- extended, terminated, expired, converted
  actor           VARCHAR(120),          -- admin user or 'system'
  meta            JSON,
  created_at      DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## API endpoints

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/admin/demo-requests/:id/provision` | Body: `{ trial_start, trial_days, modules:[group_keys], roles:[{role, seats, accounts:[{email, password}]}] }`. Validates seats ≤ cap, hashes each password, creates `trial_accounts` + `trial_modules`, sets `trial_status='active'`. |
| `POST` | `/admin/trial-accounts/:id/reset-password` | Admin sets a new password (re-hash, optionally flip `must_reset`). |
| `POST` | `/admin/trial-accounts/:id/terminate` | Sets `status='terminated'`, blocks login immediately. |
| `POST` | `/admin/demo-requests/:id/extend` | Body `{ days }`. Pushes `trial_end` + each account's `expires_at`. |
| `POST` | `/admin/demo-requests/:id/convert` | Marks `converted`, lifts caps to `requested_count`, keeps all data. |

## Auto-termination (two layers)

1. **Scheduled job** (daily, e.g. cron / Laravel scheduler):
   ```
   UPDATE trial_accounts SET status='expired'
   WHERE status = 'active' AND expires_at < NOW();
   UPDATE demo_requests SET trial_status='expired'
   WHERE trial_status='active' AND trial_end < NOW();
   ```
2. **Login guard** — on every authentication, reject if
   `status != 'active' OR expires_at < NOW()`. This protects you even if the job
   is delayed. (Don't rely on manual termination — that's how trial access leaks.)

Optional but high-value: a reminder email 1–2 days before `trial_end` — your best
moment to push a conversion.

## Status lifecycle
```
requested → provisioned → active → expired ─┐
                                  └→ converted (paid)
                                  └→ lost
```
Tracking this turns the demo list into a lightweight sales pipeline.

## How this connects to the SaaS-module work (later)
Your Step 4 groups (academic, staff, engagement, erp, ai, analytics) are already the
module families. `trial_modules` is the time-boxed version of paid module gating: when you
go SaaS, add a `school_modules` table with the same group keys plus a `plan` and date range,
and a single middleware check (`is module_group enabled for this school?`) guards every
route. Nothing about today's schema needs to change — the trial is just a 7-day row in the
same model.
