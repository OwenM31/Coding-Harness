---
name: Playwright CLI
description: Drives headless browser interactions, screenshots, and visual DOM verification via Playwright CLI and scripts
---

# Playwright CLI Skill

<role>
You are a browser automation specialist. You use Playwright via the command line and lightweight execution scripts to simulate real user interactions, capture visual screenshots, verify DOM accessibility, and detect runtime console/network errors.
</role>

---

## Core Principles

1. **Reality Over Mockery**: The browser is the ultimate arbiter of software truth. If it doesn't render in the browser, it doesn't work.
2. **Speed & Lightweight Footprint**: Use direct CLI commands and ad-hoc Node evaluation scripts over bulky end-to-end setups when performing rapid exploratory sweeps.
3. **Artifact-Driven Evidence**: Every visual assertion must be backed by a captured screenshot saved to `.agent/state/artifacts/screenshots/`.
4. **Console & Network Vigilance**: Silent JavaScript crashes (e.g., `TypeError: undefined is not a function` inside an `onClick`) must never go unnoticed.

---

## Playwright CLI Capabilities

### 1. Fast Full-Page Screenshot Capture
Capture full rendered page states across viewports without writing test suites:

```powershell
# Desktop Viewport (1280x800)
npx playwright screenshot --full-page --viewport-size=1280,800 http://localhost:5173/admin/payroll .agent/state/artifacts/screenshots/admin_payroll_desktop.png

# Mobile Viewport (iPhone 14 / 390x844)
npx playwright screenshot --full-page --viewport-size=390,844 http://localhost:5173/admin/payroll .agent/state/artifacts/screenshots/admin_payroll_mobile.png
```

### 2. PDF & Layout Export
Export printable views or check page clipping:
```powershell
npx playwright pdf http://localhost:5173/receipts/rec-001 .agent/state/artifacts/screenshots/receipt.pdf
```

### 3. Interactive Code Generation & Exploration
When an interactive trace or locator inspection is needed:
```powershell
npx playwright codegen http://localhost:5173
```

---

## Ad-Hoc Evaluation Scripts (Node + Playwright)

When CLI flags alone cannot perform multi-step interactions (e.g., logging in, clicking buttons, testing modals), run a targeted lightweight script via `node` from the project's scratch space.

### Template: Interactive Page Sweep Script
```typescript
// scratch/sweep_page.js
const { chromium } = require('@playwright/test');
const fs = require('fs');

(async () => {
  const browser = await chromium.launch({ headless: true });
  const context = await browser.newContext({
    viewport: { width: 1280, height: 800 }
  });
  const page = await context.newPage();

  // 1. Error Monitoring
  const consoleErrors = [];
  const networkErrors = [];

  page.on('console', msg => {
    if (msg.type() === 'error') consoleErrors.push(msg.text());
  });
  page.on('pageerror', error => {
    consoleErrors.push(error.message);
  });
  page.on('response', res => {
    if (res.status() >= 400) {
      networkErrors.push(`${res.status()} ${res.url()}`);
    }
  });

  // 2. Navigation
  const targetUrl = process.argv[2] || 'http://localhost:5173';
  await page.goto(targetUrl, { waitUntil: 'networkidle', timeout: 15000 });

  // 3. Inventory Interactive Elements
  const buttonElements = await page.locator('button:visible').all();
  const deadButtons = [];

  for (const btn of buttonElements) {
    const text = (await btn.innerText()).trim() || (await btn.getAttribute('aria-label')) || 'Unnamed Button';
    const beforeUrl = page.url();
    
    // Attempt non-navigating click test
    try {
      await btn.click({ timeout: 1500 });
      // Check if modal or drawer opened
      const hasModal = await page.locator('[role="dialog"]:visible').count();
      if (hasModal > 0) {
        // Close modal via Escape
        await page.keyboard.press('Escape');
      }
    } catch (e) {
      deadButtons.push({ text, error: e.message });
    }
  }

  // 4. Capture Final Visual State
  const outPath = process.argv[3] || '.agent/state/artifacts/screenshots/sweep_output.png';
  await page.screenshot({ path: outPath, fullPage: true });

  // 5. Output JSON Summary
  const report = {
    url: targetUrl,
    totalButtons: buttonElements.length,
    deadButtons,
    consoleErrors,
    networkErrors
  };

  console.log(JSON.stringify(report, null, 2));
  await browser.close();
})();
```

---

## Authentication & Session Injection

For routes gated by login or role-based access control (RBAC):

1. **Storage State Reuse**:
   If Playwright test fixtures have already authenticated, use `--load-storage`:
   ```powershell
   npx playwright screenshot --load-storage=.auth/admin.json http://localhost:5173/admin/payroll .agent/state/artifacts/screenshots/admin.png
   ```

2. **Cookie / LocalStorage Pre-Seeding**:
   Inject session tokens directly in the Node runner:
   ```javascript
   await context.addCookies([
     { name: 'st_at', value: token, domain: 'localhost', path: '/' }
   ]);
   ```

---

## Fallback & Graceful Degradation

If the target project does not have Playwright installed:
1. **Check package.json**: Look for `@playwright/test` or `playwright`.
2. **Auto-Install Option**: Run `npm install -D @playwright/test && npx playwright install chromium` if permissible.
3. **Static Fallback**: If browser execution cannot be established (e.g., pure backend environment), notify the user and fall back to static route-tree and JSX/template analysis.
