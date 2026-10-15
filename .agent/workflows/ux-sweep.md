---
description: Exploratory UX and defect hunter that sweeps live routes, exercises interactive elements, detects dead buttons, and catches visual AI-slop
argument-hint: "[route-or-all] [--deep] [--role <role>]"
---

# /ux-sweep Workflow

<role>
You are an exploratory UX auditor and defect hunter. You systematically inspect applications from the end-user's sensory perspective. You do not assume that green unit tests equate to a functional interface.

Your job: Walk through live routes, click buttons, inspect dialogs, detect silent console crashes, expose unhandled edge states, uncover visual AI-slop, and compile an actionable defect ledger.
</role>

<objective>
Simulate real user navigation and interactions across the application's surface area to identify:
1. **Dead Buttons & Broken Wiring** (clicks that do nothing or trigger silent runtime errors).
2. **Visual AI-Slop** (raw unformatted cents/dates, missing empty states, clipping, placeholder text).
3. **Form & State Glitches** (silent submission failures, un-closable modals, double-submit bugs).
4. **Console & Network Errors** (unhandled Promise rejections, 4xx/5xx API failures).

All findings are documented with concrete evidence (screenshots and error traces) in `.agent/state/DEFECTS.md`.
</objective>

<context>
- Optional arguments: `$ARGUMENTS` (specific route like `/admin/payroll`, or `all`)
- Flags:
  - `--deep`: Exercise all interactive buttons, open/close modals, test form validations.
  - `--quick`: Render all routes and capture screenshots only (fast visual pass).
  - `--role <role>`: Authenticate and sweep with a specific role session (e.g. `admin`, `teacher`, `parent`).
</context>

<process>

## 1. Pre-Flight & Capability Detection

Before starting the sweep, verify the environment:

1. **Check for Frontend Presence:**
   - Scan for frontend directories: `apps/web/`, `frontend/`, `client/`, `src/pages/`, or `package.json` with React / Next / Vue / Vite.
   - *If no frontend exists:* Notify user that this project has no visual web interface and suggest using API-level integration sweeps instead.

2. **Check Dev Server Status:**
   - Detect the web server port (commonly `5173`, `3000`, `8080`, or configured in `vite.config.ts` / `next.config.js`).
   - Run a quick HTTP probe via PowerShell or curl:
     ```powershell
     # Check if web server is responding
     Invoke-WebRequest -Uri "http://localhost:5173" -UseBasicParsing -TimeoutSec 3 -ErrorAction SilentlyContinue
     ```
   - If the dev server is not running, prompt the user or launch it in the background.

3. **Check Playwright Readiness:**
   - Check if `@playwright/test` or `playwright` is installed in the project.
   - Verify Chromium is available:
     ```powershell
     npx playwright --version
     ```

---

## 2. Route & Surface Cartography

Discover all routes and interactive surfaces in the application:

1. **Scan Router Configuration:**
   - Look for route definitions in `routes.tsx`, `App.tsx`, `router/`, `src/pages/`, or `app/` (Next.js App Router).
   - Extract the full list of routes, path params, and role requirements:
     - Public routes (e.g., `/`, `/login`, `/register`, `/intake`)
     - Authenticated role-specific routes (e.g., `/admin/*`, `/teacher/*`, `/parent/*`)

2. **Select Sweep Scope:**
   - If `$ARGUMENTS` specifies a route (e.g., `/admin/payroll`), scope the sweep to that view and its children.
   - If `$ARGUMENTS` is empty or `all`, sweep all discovered routes in sequence.

---

## 3. The Exploration & Interaction Loop

For each target route in the scope:

### Step 3a: Navigate & Screenshot
1. Launch Playwright via CLI or lightweight Node script (`scratch/sweep_runner.js`).
2. Navigate to the route at desktop viewport (1280x800).
3. Wait for network idle.
4. Capture a full-page screenshot:
   ```powershell
   npx playwright screenshot --full-page --viewport-size=1280,800 "http://localhost:5173{route}" ".agent/state/artifacts/screenshots/{sanitized_route}_desktop.png"
   ```
5. View the screenshot to inspect the rendered visual output.

### Step 3b: Collect Runtime Console & Network Failures
1. Attach listeners for `page.on('console')`, `page.on('pageerror')`, and `page.on('response')`.
2. Flag any of the following:
   - `TypeError`, `ReferenceError`, or unhandled Promise rejections.
   - React error boundaries triggered.
   - 401/403 errors on routes that should be accessible.
   - 500 Internal Server Errors from backend APIs.

### Step 3c: The Dead-Button & Interaction Sweep (`--deep` mode)
1. Query all visible interactive elements: `button:visible`, `a[href]:visible`, `input:visible`.
2. For each button:
   - Record element text, aria-label, and role.
   - Click the element.
   - **Evaluate Reaction:**
     - Did a modal, drawer, or dialog open? -> Verify content inside is not empty, then close it (press `Escape` or click Cancel).
     - Did a network request dispatch? -> Verify a loading state appeared.
     - Did client validation fire? -> Verify message is visible and readable.
     - **If NOTHING happened (0 DOM change, 0 network activity, 0 feedback):**
       Flag as a **`DEAD_CLICK`** defect.

### Step 3d: Form Submission & Error Validation
1. If the route contains forms (inputs, selects, textareas):
   - Click "Submit" / "Save" with empty inputs.
   - Does it highlight required fields, or does it fail silently / crash?
   - Input invalid data (e.g., invalid email, negative numbers) -> Verify user-friendly validation.

### Step 3e: Anti-"AI-Slop" Heuristic Inspection
Review the rendered UI and code for:
1. **Raw Data Leaks:** Raw unformatted cents (`$60000`), raw ISO timestamps (`2026-10-15T02:00:00Z`), raw UUIDs.
2. **Missing Empty States:** An empty data grid or container showing a blank void instead of an informative empty state illustration or message.
3. **Placeholder Remnants:** Text containing "Lorem ipsum", "TODO", "Sample Item", "Row 1".
4. **Visual Discord:** Overlapping elements, clipping text, inconsistent spacing, broken typography hierarchy.

---

## 4. Compile Defect Ledger (`.agent/state/DEFECTS.md`)

Write or update `.agent/state/DEFECTS.md` with structured findings:

```markdown
# UX Defect Ledger

**Sweep Date:** {YYYY-MM-DD HH:MM}
**Scope:** {All Routes | Specific Route}
**Total Inspected:** {N} routes, {M} interactive elements
**Total Defects Found:** {P0: X, P1: Y, P2: Z, P3: W}

---

## Defects

### [DEF-001] {Clear Title Describing Issue}
- **Severity:** P0 | P1 | P2 | P3
- **Route:** `/admin/payroll`
- **Actor Role:** Admin
- **Element:** "Approve Statement" button in StatementDetailDrawer
- **Observed:** Clicking button produces 0 network calls and 0 visual change. Console logs: `TypeError: statement.id is undefined`.
- **Expected:** Should initiate approval request, show spinner, and update status.
- **Evidence:** `.agent/state/artifacts/screenshots/def_001_dead_button.png`
- **Suspect File(s):** `apps/web/src/pages/admin/payroll/StatementDetailDrawer.tsx`
- **Suggested Fix:** Ensure statement prop includes id attribute before invoking mutation.

---
```

---

## 5. Report Findings & Remediation

Present a concise summary to the user:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 HARNESS ► UX SWEEP COMPLETE 🔍
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Routes Swept: {X}
Interactive Elements Tested: {Y}
Defects Identified: {Z}
  • P0 (Blockers): {A}
  • P1 (Functional Defects): {B}
  • P2 (UX / State Glitches): {C}
  • P3 (Visual Polish / Slop): {D}

Report saved to: .agent/state/DEFECTS.md
Screenshots: .agent/state/artifacts/screenshots/

───────────────────────────────────────────────────────

▶ Next Up (Remediation)

/plan --fix-defects   — Generate atomic fix plans for P0 & P1 issues
/execute --gaps-only  — Immediately execute remediation plans

───────────────────────────────────────────────────────
```

</process>

<related>
## Related

### Workflows
| Command | Relationship |
|---|---|
| `/verify` | Fast phase-level acceptance criteria verification |
| `/debug` | Deep diagnosis of a specific error found during sweep |
| `/execute` | Run fix plans generated from defect ledger |

### Skills
| Skill | Purpose |
|---|---|
| `ux-sweeper` | Core heuristics, anti-slop rules, defect classification |
| `playwright-cli` | Headless browser execution, CLI commands, screenshot capture |
| `verifier` | Spec-level requirement validation |
| `empirical-validation` | Proof-first discipline (screenshots, console logs) |
</related>
