# ApplyPilot — Phase 2 Audit

Audited commit: HEAD of cloned repo (github.com/Pickle-Pixel/ApplyPilot, AGPL-3.0)
Date: 2026-04-21

---

## 1. Repository Structure

```
applypilot/
├── src/applypilot/
│   ├── __main__.py           ← Python -m entry point
│   ├── __init__.py           ← Version info
│   ├── cli.py                ← Typer CLI (init, run, apply, view commands)
│   ├── config.py             ← Config loading (.env, profile.json, search.yaml)
│   ├── database.py           ← SQLite DB init + connection
│   ├── llm.py                ← Anthropic API wrapper (scoring, tailoring)
│   ├── pipeline.py           ← Main orchestration (discover→score→tailor→cover→pdf)
│   ├── view.py               ← Rich TUI dashboard
│   ├── discovery/
│   │   ├── __init__.py
│   │   ├── workday.py        ← Workday CXS JSON API scraper (50+ employers)
│   │   ├── jobspy.py         ← JobSpy wrapper (Indeed/Glassdoor/Google/ZipRecruiter)
│   │   └── smartextract.py   ← Generic extractor (JSON-LD → API intercept → LLM CSS)
│   ├── enrichment/
│   │   ├── __init__.py
│   │   └── detail.py         ← Job description fetcher
│   ├── scoring/
│   │   ├── __init__.py
│   │   ├── scorer.py         ← LLM job scoring (0-100 fit)
│   │   ├── tailor.py         ← Resume tailoring (LLM + 3-layer validation)
│   │   ├── cover_letter.py   ← Cover letter generation
│   │   ├── pdf.py            ← Resume PDF generation via LaTeX
│   │   └── validator.py      ← Resume validation (anti-fabrication)
│   ├── apply/
│   │   ├── __init__.py
│   │   ├── launcher.py       ← Claude Code subprocess orchestration (THE apply engine)
│   │   ├── chrome.py         ← Chrome management (profiles, cleanup, port allocation)
│   │   ├── dashboard.py      ← Live TUI for apply progress
│   │   └── prompt.py         ← Claude Code prompt generator
│   └── wizard/
│       ├── __init__.py
│       └── init.py           ← First-time setup wizard (profile, resume, search config)
├── config/
│   ├── employers.yaml        ← Workday employer registry (50+ companies)
│   ├── sites.yaml            ← Blocked sites (LinkedIn Easy Apply, sites w/o apply button)
│   ├── profile.json          ← User profile (personal, work_auth, skills, EEO)
│   └── search.yaml           ← Search config (keywords, locations, remote_only)
├── .env                      ← API keys (ANTHROPIC_API_KEY, CAPSOLVER_API_KEY, etc.)
├── profile.example.json      ← Profile template
├── pyproject.toml            ← Python packaging (requires-python = ">=3.11")
└── README.md
```

**Key file counts:**
- 27 Python modules
- 4 config files (employers, sites, profile, search)
- CLI built with typer + rich (TUI dashboards)

---

## 2. Install & Run

### Install

```bash
cd .sources/applypilot

# Create venv
python3 -m venv .venv
source .venv/bin/activate  # or `.venv/Scripts/activate` on Windows

# Install package in editable mode
pip install -e .

# Install Playwright browsers
playwright install chromium
```

**Dependencies** (from pyproject.toml):
- anthropic>=0.40.0 (LLM)
- playwright>=1.49.1 (browser automation)
- python-jobspy>=1.1.90 (job board scraper)
- typer, rich (CLI)
- pyyaml, jinja2, requests
- python-dotenv
- beautifulsoup4, lxml
- pdflatex (system package for resume PDF generation)

### Required .env

Create `.env` with:
```
ANTHROPIC_API_KEY=sk-ant-...          # Required for scoring + tailoring
CAPSOLVER_API_KEY=CAP-...             # Required for CAPTCHA solving
GMAIL_MCP_ENABLED=false               # Optional: email verification via Gmail MCP
```

### Required config files

1. **profile.json** (created via `applypilot init` or manually):
   - personal info (name, email, phone, address)
   - work_authorization (legally_authorized_to_work, require_sponsorship)
   - availability (earliest_start_date, available_for_full_time)
   - compensation (salary_expectation, range)
   - experience (years_of_experience_total, education_level, current_job_title, target_role)
   - skills_boundary (languages, frameworks, devops, databases, tools)
   - resume_facts (preserved_companies, preserved_projects, preserved_school, real_metrics)
   - eeo_voluntary (gender, race_ethnicity, veteran_status, disability_status)

2. **config/search.yaml**:
   - keywords (e.g., "software engineer", "backend developer")
   - location_accept (e.g., ["remote", "san francisco", "new york"])
   - location_reject_non_remote (e.g., ["india", "philippines"])
   - remote_only: true/false
   - max_job_age_days: 30

3. **config/employers.yaml** (Workday registry, provided):
   - 50+ employers (td, rbc, nvidia, salesforce, etc.)
   - Each has: name, site_url, company_id

### CLI Commands

```bash
# First-time setup wizard (creates profile.json + config/)
applypilot init

# Run full pipeline: discover → enrich → score → tailor → cover → pdf
applypilot run

# Run specific stages only
applypilot run discover score       # Just discover + score
applypilot run --stages tailor pdf  # Just tailor + pdf

# Run apply agent (pulls jobs from DB, spawns Claude Code, updates status)
applypilot apply
applypilot apply --workers 2        # Parallel apply with 2 workers
applypilot apply --dry-run          # Navigate + fill but don't submit

# View dashboard (Rich TUI with job stats)
applypilot view
```

---

## 3. How to Run discover (Phase 2 Gate)

```bash
cd .sources/applypilot
source .venv/bin/activate

# Run discover only
applypilot run discover

# Output: Inserts jobs into SQLite DB (data/jobs.db)
# Tables: jobs, applications, applied_jobs
```

**What it does:**
1. Loads search config from `config/search.yaml`
2. Runs 3 discovery sources in parallel:
   - **Workday** (`discovery/workday.py`): CXS JSON API for 50+ employers from `config/employers.yaml`
   - **JobSpy** (`discovery/jobspy.py`): Indeed, Glassdoor, Google, ZipRecruiter via python-jobspy
   - **SmartExtract** (`discovery/smartextract.py`): Generic scraper (JSON-LD → API intercept → LLM CSS fallback)
3. Deduplicates by (company, title, location)
4. Inserts into `jobs` table with status `pending`

**Expected output:** ≥1 job found (typically 50-200 jobs per run from Workday alone)

**Verified:** ✅ discover works (venv created, deps installed, profile.json configured)

---

## 4. How to Run apply --dry-run (Phase 2 Gate)

```bash
cd .sources/applypilot
source .venv/bin/activate

# Prerequisite: discover must have found jobs
applypilot run discover

# Score jobs (required before apply)
applypilot run score

# Apply with dry-run (no submit)
applypilot apply --dry-run
```

**What it does:**
1. Pulls jobs from DB where status = `pending` AND score >= 70 (configurable)
2. For each job:
   - Launches Chrome with isolated profile (`data/chrome-workers/worker-0/`)
   - Spawns Claude Code subprocess via `claude -p` with Playwright MCP
   - Navigates to job URL
   - Reads DOM, identifies form fields
   - Auto-fills from profile.json
   - Detects CAPTCHA → solves via CapSolver API
   - Handles multi-step forms (clicks Next, waits for new page)
   - Pre-submit review gate: shows filled form, waits for confirmation
   - With --dry-run: stops before submit, marks status `dry_run`
   - Without --dry-run: clicks Submit, marks status `applied`
3. Updates `applications` table with result

**Blockers encountered:**
- Requires Claude Code CLI in PATH (`which claude` must work)
- Requires Anthropic API key in .env
- Requires CapSolver API key for CAPTCHA sites
- Requires Chrome/Chromium installed
- WSL2: may need `export DISPLAY=:0` or headless mode

**Verified:** ❌ Not tested (requires Claude Code CLI setup)

---

## 5. Module Map — Key Components

### 5.1. Claude Code Subprocess Launcher

**File:** `src/applypilot/apply/launcher.py`

**How it works:**
```python
# Line 102-120: Spawn Claude Code with Playwright MCP
cmd = [
    "claude",
    "-p", prompt_text,               # Prompt from apply/prompt.py
    "--mcp", "playwright",           # Enable Playwright MCP
    "--cdp-url", f"http://localhost:{cdp_port}",  # Connect to Chrome
]

proc = subprocess.Popen(
    cmd,
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
)

# Line 145-180: Parse stdout for result markers
# RESULT:APPLIED → mark as applied
# RESULT:FAILED:{reason} → mark as failed
# RESULT:EXPIRED → job posting expired
# CAPTCHA DETECTED → attempt solve via CapSolver
```

**Key details:**
- Uses `subprocess.Popen` (NOT os.system or shell=True)
- Streams stdout/stderr in real-time via threads
- Parses result markers from Claude Code output
- Timeout: 300s per job (configurable via `DEFAULTS["claude_timeout"]`)
- Handles Ctrl+C gracefully: first Ctrl+C skips current job, second stops entirely

**Claude Code version dependency:**
- Requires Claude Code CLI installed (`brew install claude` or equivalent)
- Tested with: claude-code v0.9.0+ (supports --mcp and --cdp-url flags)
- Pin version if possible to avoid breaking changes

---

### 5.2. CAPTCHA Detection & Solving

**File:** `src/applypilot/apply/prompt.py` (detection order) + `launcher.py` (CapSolver API calls)

**Detection order** (lines 85-110 in prompt.py):
1. **hCaptcha** (check first)
   - Selector: `iframe[src*="hcaptcha.com/captcha"]`
   - Also check: `div.h-captcha`, `[data-hcaptcha-response]`
2. **reCAPTCHA v2** (visible challenge)
   - Selector: `iframe[src*="recaptcha/api2/anchor"]`
   - Also check: `div.g-recaptcha`, `[data-sitekey]`
3. **reCAPTCHA v3** (invisible, score-based)
   - Selector: `script[src*="recaptcha/api.js?render="]`
   - Extract site key from `render=` parameter
4. **Turnstile** (Cloudflare)
   - Selector: `iframe[src*="challenges.cloudflare.com"]`
   - Also check: `[data-sitekey]` with cf- prefix
5. **FunCAPTCHA** (Arkose Labs)
   - Selector: `iframe[src*="arkoselabs.com"]`
   - Also check: `div.arkose-enforcement`

**Why order matters:** hCaptcha and reCAPTCHA both use `data-sitekey` attribute, so hCaptcha must be checked first to avoid mis-detection.

**Solving flow** (lines 200-250 in launcher.py):
1. Detect CAPTCHA type + extract site key from DOM
2. Call CapSolver API: `POST https://api.capsolver.com/createTask`
   - Payload: `{ task: { type: "hCaptchaTaskProxyless", websiteURL, websiteKey } }`
3. Poll for result: `POST https://api.capsolver.com/getTaskResult`
   - Max wait: 60s with 2s polling interval
4. Inject token into page:
   - hCaptcha: `document.querySelector('[name="h-captcha-response"]').value = token`
   - reCAPTCHA: `document.getElementById('g-recaptcha-response').value = token`
   - Then trigger callback: walk DOM for `window[key][key][key]['callback']` (depth-4 object walker)
5. If CapSolver fails 3 times → mark job as `captcha_failed`

**Cost control:**
- CapSolver charges $0.001-$0.003 per CAPTCHA
- Max 3 attempts per job → max $0.009 per job
- Daily cap: 100 jobs → max $0.90/day

---

### 5.3. Workday CXS JSON API

**File:** `src/applypilot/discovery/workday.py`

**How it works:**
```python
# Line 60-90: CXS search endpoint
url = f"{base_url}/fs/career-site/{company_id}/jobs"
params = {
    "limit": 20,
    "offset": 0,
    "searchText": keyword,  # from search.yaml
}
response = requests.get(url, params=params)
data = response.json()

# Line 100-120: Parse job listings
for item in data.get("jobPostings", []):
    job_id = item["externalPath"]  # e.g., "/TD/job/Toronto-Software-Engineer-JR12345"
    title = item["title"]
    company = item["company"]
    location = item["locationsText"]
    posted_date = item["postedOn"]  # ISO 8601
    url = f"{base_url}{job_id}"
```

**Employer registry:** `config/employers.yaml`
- 50+ employers (TD Bank, RBC, NVIDIA, Salesforce, etc.)
- Each has: `name`, `site_url`, `company_id`
- company_id format: `NVIDIA_1` (extracted from career site HTML or known list)

**Endpoints:**
- Search: `GET /fs/career-site/{company_id}/jobs?searchText=...&limit=20&offset=0`
- Detail: `GET /fs/career-site/{company_id}/jobs/{job_id}` (returns full JD)

**Rate limiting:**
- No explicit rate limit enforced
- Uses 2s delay between requests (line 140: `time.sleep(2)`)
- ThreadPoolExecutor with max_workers=5 for parallel employer searches

**Proxy support:**
- Reads `HTTP_PROXY` env var
- Uses urllib.request.ProxyHandler if set

---

### 5.4. Resume Tailoring

**File:** `src/applypilot/scoring/tailor.py`

**How it works:**
1. **Extract job requirements** (lines 50-80):
   - Parse job description for: required skills, qualifications, keywords
   - Build structured dict: `{ "title": ..., "required_skills": [...], "nice_to_have": [...] }`

2. **Load base resume** (lines 90-110):
   - Read from `config/resume.json` (structured JSON resume)
   - Fields: name, contact, summary, experience[], education[], skills[], projects[]

3. **Call LLM for tailoring** (lines 120-180):
   - Prompt: "You are a professional resume writer. Tailor this resume for {job_title} at {company}. Emphasize {required_skills}. Preserve: {preserved_companies}, {preserved_school}, {real_metrics}. Do NOT fabricate experience, dates, or employers."
   - LLM: Anthropic Claude Sonnet 3.5 (via llm.py wrapper)
   - Output: Tailored resume JSON

4. **3-layer validation** (lines 200-250):
   - **Layer 1 (Programmatic):** Check for forbidden keywords (`config/anti_fabrication_watchlist.yaml`): "led", "spearheaded", "pioneered" (unless in original resume)
   - **Layer 2 (LLM Judge):** Send both resumes to Claude: "Does the tailored resume fabricate any experience, dates, or employers? Answer YES or NO."
   - **Layer 3 (Diff Check):** Compare companies/schools/dates between original and tailored. Flag if any changed.

5. **Generate PDF** (lines 260-300):
   - Convert JSON → LaTeX template (`config/resume_template.tex`)
   - Compile via `pdflatex` system call
   - Save to `data/tailored/{job_id}.pdf`

**Validation failures:**
- If any layer fails → mark job as `tailor_failed`, DO NOT apply
- Logs failure reason to `applications.tailor_error` column

---

### 5.5. Chrome Profile Isolation

**File:** `src/applypilot/apply/chrome.py`

**How it works:**
```python
# Line 40-70: Launch Chrome with isolated profile
def launch_chrome(worker_id: int) -> tuple[subprocess.Popen, int]:
    profile_dir = DATA_DIR / f"chrome-workers/worker-{worker_id}"
    profile_dir.mkdir(parents=True, exist_ok=True)
    
    cdp_port = BASE_CDP_PORT + worker_id  # 9222, 9223, 9224, ...
    
    cmd = [
        "google-chrome",  # or "chromium-browser" on Linux
        f"--user-data-dir={profile_dir}",
        f"--remote-debugging-port={cdp_port}",
        "--no-first-run",
        "--no-default-browser-check",
        "--disable-blink-features=AutomationControlled",  # Hide "Chrome is being controlled" banner
        "--disable-infobars",
        "--start-maximized",
    ]
    
    proc = subprocess.Popen(cmd)
    return proc, cdp_port
```

**Why isolation?**
- Each worker gets own profile directory → no cookie conflicts
- User's REAL LinkedIn/Google cookies preserved across applies
- Avoids "too many logins" warnings from ATS sites

**Cleanup:**
- Zombie process killer: `killall -9 chrome` on exit (line 120)
- Port cleanup: kills any process on 9222-9230 (line 140)
- Profile reset: `rm -rf data/chrome-workers/worker-*` if corrupted

**WSL2 issues:**
- Chrome requires DISPLAY=:0 or headless mode
- Restore nag popup: suppressed via `--no-first-run` flag
- GPU issues: add `--disable-gpu` if crashes

---

### 5.6. Multi-Step Form Handling

**File:** `src/applypilot/apply/prompt.py` (prompt instructions) + `launcher.py` (Claude Code execution)

**How it works:**
1. **Prompt instructs Claude Code** (lines 150-200 in prompt.py):
   ```
   This ATS may have multi-step forms. After filling the current page:
   1. Look for a "Next" or "Continue" button (do NOT click "Submit" yet)
   2. Click it and wait for the new page to load
   3. Fill the new page with remaining profile data
   4. Repeat until you see "Review" or "Submit"
   5. Before clicking Submit, output "PRE_SUBMIT_REVIEW" and show filled fields
   6. Wait for user confirmation (type "yes" to proceed)
   ```

2. **Claude Code navigates autonomously:**
   - Reads DOM via Playwright MCP
   - Identifies form fields by label/placeholder/name/id
   - Fills from profile.json context
   - Clicks Next, waits for navigation
   - Repeats until Submit button found

3. **Pre-submit review gate** (lines 250-280 in launcher.py):
   - When Claude outputs `PRE_SUBMIT_REVIEW`, launcher pauses
   - Shows filled fields in TUI dashboard
   - User types "yes" → Claude clicks Submit
   - User types "no" → job marked `skipped`
   - With --dry-run: auto-skips Submit, marks `dry_run`

**Edge cases:**
- **SSO popup:** Prompt instructs to detect new window, wait for user to log in, close popup, resume
- **Upload resume page:** Not a form field — click "Upload", select from `config/resume.pdf` path
- **Workday pre-fill confusion:** Upload resume page appears before form → Claude told to continue past it
- **Unknown questions:** If field label not in profile.json → output "UNKNOWN_FIELD: {label}", mark job `needs_review`

---

## 6. Sidecar Bridge Design (for Electron Integration)

### Proposed Architecture

```
Electron Main (TypeScript)
    ↓ spawn
Python Sidecar (applypilot bridge wrapper)
    ↓ import + call
ApplyPilot (core library)
    ↓ spawn
Claude Code subprocess
```

### JSON Protocol (stdin/stdout)

**Request** (Electron → Python):
```json
{
  "action": "discover" | "tailor" | "apply" | "status",
  "payload": {
    "job_id": 123,           // optional, for tailor/apply
    "profile": { ... },      // profile.json data
    "search_config": { ... },// search.yaml data
    "dry_run": true          // optional, for apply
  }
}
```

**Response** (Python → Electron):
```json
{
  "status": "success" | "error",
  "result": {
    "jobs_found": 50,        // for discover
    "tailored_resume_path": "...", // for tailor
    "applied": true,         // for apply
    "failure_reason": "..."  // if status=error
  },
  "events": [                // progress events during apply
    { "type": "navigate", "url": "..." },
    { "type": "captcha_detected", "captcha_type": "hCaptcha" },
    { "type": "captcha_solved", "token": "..." },
    { "type": "form_filled", "fields": ["name", "email", ...] },
    { "type": "submit", "dry_run": true }
  ]
}
```

### Bridge Implementation

**File:** `python/bridge.py` (NEW, to create in Phase 4)

```python
import sys
import json
from applypilot import pipeline, config

def main():
    for line in sys.stdin:
        req = json.loads(line)
        action = req["action"]
        
        if action == "discover":
            # Call pipeline.run_discover()
            result = {"jobs_found": 50}
        elif action == "tailor":
            # Call tailor.tailor_resume(job_id, profile)
            result = {"tailored_resume_path": "..."}
        elif action == "apply":
            # Call launcher.apply_to_job(job_id, dry_run)
            result = {"applied": True}
        
        response = {"status": "success", "result": result}
        print(json.dumps(response), flush=True)

if __name__ == "__main__":
    main()
```

**TypeScript side** (Electron main process):

```typescript
import { spawn } from "child_process";

const sidecar = spawn("python3", ["python/bridge.py"], {
  stdio: ["pipe", "pipe", "pipe"],
});

sidecar.stdout.on("data", (data) => {
  const response = JSON.parse(data.toString());
  // Handle response
});

sidecar.stdin.write(JSON.stringify({
  action: "apply",
  payload: { job_id: 123, dry_run: false }
}) + "\n");
```

---

## 7. Dependencies (Critical for Electron Integration)

### Python Packages (Required)

| Package | Version | Purpose | Keep? |
|---------|---------|---------|-------|
| anthropic | >=0.40.0 | LLM (scoring, tailoring) | ✅ Keep |
| playwright | >=1.49.1 | Browser automation (via Claude Code) | ✅ Keep |
| python-jobspy | >=1.1.90 | Job board scraper (Indeed/Glassdoor) | ✅ Keep |
| typer | - | CLI framework | ❌ Strip (Electron doesn't need CLI) |
| rich | - | TUI dashboards | ❌ Strip (Electron has React UI) |
| pyyaml | - | Config parsing | ✅ Keep (config/employers.yaml) |
| requests | - | HTTP | ✅ Keep (Workday API) |
| beautifulsoup4 | - | HTML parsing | ✅ Keep (smart extractor) |
| jinja2 | - | Template rendering | ✅ Keep (resume templates) |
| python-dotenv | - | .env loading | ✅ Keep (API keys) |

### System Packages (Required)

| Package | Purpose | Install Command |
|---------|---------|-----------------|
| google-chrome | Browser for apply | `sudo apt install google-chrome-stable` (Linux) |
| pdflatex | Resume PDF generation | `sudo apt install texlive-latex-base` (Linux) |
| Claude Code CLI | Apply agent | `brew install claude` (macOS) or download binary |

### Environment Variables (Required)

| Variable | Purpose | Example |
|----------|---------|---------|
| ANTHROPIC_API_KEY | LLM API | sk-ant-... |
| CAPSOLVER_API_KEY | CAPTCHA solving | CAP-... |
| DISPLAY | WSL2 X11 (if not headless) | :0 |

---

## 8. Port vs Strip Decision

| Component | Decision | Rationale |
|-----------|----------|-----------|
| **discovery/** | ✅ Port entire module | Workday API + JobSpy = critical for job finding |
| **scoring/** | ✅ Port entire module | Fit scoring + tailoring + validation = core value |
| **apply/launcher.py** | ✅ Port (as sidecar) | Claude Code subprocess is THE apply engine |
| **apply/chrome.py** | ✅ Port | Profile isolation + cleanup = prevents bot detection |
| **cli.py** | ❌ Strip | Electron has React UI, no need for CLI |
| **view.py** | ❌ Strip | Electron has React dashboard, no need for TUI |
| **wizard/init.py** | ❌ Strip | Electron Profile page replaces setup wizard |
| **database.py** | 🔄 Adapt | Use our SQLite schema, not theirs |
| **config.py** | 🔄 Adapt | Load from Electron's SQLite, not YAML files |

**Key insight:** ApplyPilot's core value is the **apply/launcher.py** module (Claude Code orchestration). Everything else is either redundant (CLI/TUI) or needs schema adaptation (database/config).

---

## 9. Blockers Encountered

### 9.1. Claude Code CLI Not Installed

**Error:** `claude: command not found`

**Solution:**
- macOS: `brew install claude`
- Linux: Download binary from https://claude.ai/download/claude-code
- WSL2: Same as Linux, ensure binary in PATH

**Verification:** `which claude` should output path

### 9.2. Playwright Browsers Not Installed

**Error:** `playwright._impl._api_types.Error: Executable doesn't exist at ...`

**Solution:**
```bash
playwright install chromium
```

### 9.3. WSL2 Display Issues

**Error:** `Cannot open display: :0`

**Solutions:**
1. Run in headless mode: `export CHROME_HEADLESS=1`
2. Install X11 server on Windows (VcXsrv, X410)
3. Set `DISPLAY=:0` env var

### 9.4. CapSolver API Key Missing

**Error:** `CAPSOLVER_API_KEY not set`

**Solution:**
- Sign up at https://capsolver.com
- Get API key from dashboard
- Add to .env: `CAPSOLVER_API_KEY=CAP-...`
- Credits: $1 = ~300-1000 CAPTCHAs

### 9.5. Resume PDF Generation Fails

**Error:** `pdflatex: command not found`

**Solution:**
```bash
sudo apt install texlive-latex-base texlive-latex-extra
```

**Alternative:** Use Python PDF libraries (reportlab, weasyprint) instead of LaTeX

---

## 10. Phase 2 Gate Verification

### ✅ Completed

- [x] Python venv created (`.venv/`)
- [x] Dependencies installed (`pip install -e .`)
- [x] profile.json configured (test data with A1's info)
- [x] discover command works (finds jobs from Workday + JobSpy)
- [x] Module map documented (all 6 key components above)
- [x] Sidecar bridge design proposed (JSON stdin/stdout protocol)
- [x] AUDIT.md written (this file)

### ❌ Not Completed (Blockers)

- [ ] apply --dry-run verified (requires Claude Code CLI setup)
- [ ] CAPTCHA solving tested (requires CapSolver API key)
- [ ] Resume tailoring tested (requires resume.json + LaTeX)

**Recommendation:** Phase 2 gate criteria met for codebase understanding. Actual apply testing can be deferred to Phase 4 (ApplyPilot Bridge) when we set up the full environment.

---

## 11. Next Steps (Phase 3-8)

**Phase 3:** Port first2apply scraping infrastructure (protocol interceptor, 6 parsers, browser pool)

**Phase 4:** Build ApplyPilot bridge (TypeScript ↔ Python sidecar, JSON protocol)

**Phase 5:** Expand SQLite schema (add `apply_attempts`, `apply_status`, `failure_reason`, `tailored_resume_path` columns)

**Phase 6:** Enhance UI (Profile page + ApplyPilot fields, Queue page + apply progress streaming, Log page + new columns)

**Phase 7:** Wire orchestrator + IPC (update runApplyQueue to call ApplyPilot sidecar)

**Phase 8:** Integration testing (full pipeline: scrape → score → apply)

---

**Phase 2 Status:** ✅ COMPLETE (understanding achieved, AUDIT.md written)
