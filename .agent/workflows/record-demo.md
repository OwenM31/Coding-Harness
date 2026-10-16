---
description: Record interactive user flows with Playwright, automatic user story annotations (HUD), and high-craft GIF optimization for READMEs
argument-hint: "[flow-or-story] [--url <url>] [--story <name>] [--hud bottom-left|bottom-right|top-left|top-right|off] [--output <path>] [--fps <10-20>] [--width <pixels>]"
---

# /record-demo Workflow

<role>
You are an elite technical documentation specialist and browser automation recording engineer. You choreograph, capture, and optimize smooth, annotated demonstration GIFs of user flows to showcase in READMEs, PRs, and documentation.

Your job: Ingest the target user story or acceptance criteria, configure human-paced choreography with an in-browser HUD overlay (step pill badges and click ripple indicators), record the session via Playwright, convert the resulting video into a crisp, lightweight `.gif` (< 5MB) using 2-pass color palette optimization, and output the ready-to-embed Markdown.
</role>

<objective>
Deliver an end-to-end, executive-ready demo recording pipeline:
1. **User Story & Criteria Ingestion**: Extract atomic acceptance criteria from `.agent/state/SPEC.md` (or user prompt) and map them directly into sequential demo step captions.
2. **In-Browser Glassmorphic HUD (Annotations)**: Inject a sleek floating step-by-step HUD badge (with auto-recovery across full page reloads via `addInitScript`) displaying: `Step N: <Atomic Action>`.
3. **Virtual Click Ripple Indicators**: Render subtle expanding ripple animations at the click coordinates so viewers can clearly follow cursor interactions in headless recordings.
4. **Human-Paced Choreography**: Enforce intentional visual dwell times (realistic typing speed, 600–1000ms transition pauses, and a 2.5s hold on final state before looping).
5. **Two-Pass Color Palette GIF Optimization**: Generate a custom 256-color palette to compress HD recordings into crisp, artifact-free GIFs targeting `< 5 MB`.
6. **Documentation & README Integration**: Place the winning asset into `assets/demos/` and output the exact Markdown embed snippet.
7. **Re-Recordability**: Preserve the recording script in `tests/demos/` so future UI changes can re-record the demo deterministically.
</objective>

<context>
- **Arguments:** `$ARGUMENTS` (e.g. `"checkout-flow"`, `"user registration --hud bottom-right"`, `"--story 'Search and filter products'"`)
- **Key Flags:**
  - `--url <url>`: Target start URL (default: probe local dev server e.g. `http://localhost:3000`, `http://localhost:5173`, etc.)
  - `--story <name>`: Target user story from `SPEC.md` or active phase plan
  - `--hud <pos>`: Position for the floating annotation badge: `bottom-left` (default), `bottom-right`, `top-left`, `top-right`, or `off`
  - `--output <path>`: Destination path (default: `assets/demos/<flow-slug>.gif` or `docs/demos/<flow-slug>.gif`)
  - `--fps <N>`: Frame rate for GIF (default: `15`, optimal balance of smoothness and file size)
  - `--width <px>`: Target width in pixels (default: `800` for desktop views, `400` for mobile views)
  - `--headed`: Run browser visibly for real-time observation
  - `--save-script <path>`: Destination to preserve the repeatable recording script (default: `tests/demos/<flow-slug>.demo.js`)
- **Reference Skills:**
  - `.agent/skills/playwright-cli/SKILL.md` (CLI browser automation & session control)
  - `.agent/skills/playwright-cli/references/video-recording.md` (video recording reference)
  - `.agent/skills/frontend-designer/SKILL.md` (visual standards and anti-slop rules)
</context>

<process>

## 1. Parse Arguments & Ingest User Stories

1. **Extract Arguments:**
   - Parse `$ARGUMENTS` for flow name, `--url`, `--story`, `--hud`, `--output`, `--fps`, and `--width`.

2. **Ingest User Stories / Acceptance Criteria:**
   Extract atomic criteria to use as step captions from one of three sources:
   - **Source A (SPEC.md / Phase Plan):** Search `.agent/state/SPEC.md` or the active phase plan in `.agent/state/phases/` for user stories or acceptance criteria matching `--story <name>` or the flow topic.
   - **Source B (Explicit Prompt):** If the user provided a step list in their prompt (e.g. "1. click login, 2. enter email, 3. click submit"), parse these directly.
   - **Source C (Interactive Interview):** If no steps are provided, prompt the user:
     > *"Which user story or flow should we record? (e.g., 'Filter products by price and add to cart'). What are the 3–5 key steps you want shown on screen?"*

3. **Formulate the Step Sequence Ledger:**
   Create an ordered array of atomic action steps:
   ```json
   [
     { "step": 1, "caption": "Navigate to Product Catalog", "action": "goto", "target": "/products" },
     { "step": 2, "caption": "Search for 'Wireless Headphones'", "action": "type", "selector": "[data-testid=search-input]", "value": "Wireless Headphones" },
     { "step": 3, "caption": "Apply 'Under $100' Price Filter", "action": "click", "selector": "[data-testid=filter-under-100]" },
     { "step": 4, "caption": "Add Top Result to Shopping Cart", "action": "click", "selector": "[data-testid=add-to-cart-1]" }
   ]
   ```
   Present this ledger to the user before running so they can verify the wording and flow.

---

## 2. Pre-Flight & Capability Verification

1. **Verify Dev Server Status:**
   Probe the target URL or standard local ports (`5173`, `3000`, `8080`, `4321`):

   **PowerShell:**
   ```powershell
   $testUrl = "http://localhost:5173"
   try {
       $resp = Invoke-WebRequest -Uri $testUrl -UseBasicParsing -TimeoutSec 3 -ErrorAction Stop
       Write-Host "[OK] Dev server running at $testUrl (Status: $($resp.StatusCode))" -ForegroundColor Green
   } catch {
       Write-Host "[WARN] Dev server not responding at $testUrl. Verify local server before recording." -ForegroundColor Yellow
   }
   ```

   **Bash:**
   ```bash
   curl -s -o /dev/null -w "%{http_code}" --connect-timeout 3 http://localhost:5173 || echo "Dev server not responding"
   ```

   If the server is not active, prompt the user or help launch it before proceeding.

2. **Verify Playwright & Video Conversion Engine:**
   - Ensure `@playwright/test` or `playwright` is available.
   - Test `ffmpeg -version` (primary GIF engine).
   - If `ffmpeg` is not installed, note fallback to Python (`Pillow`/`moviepy`) or provide copy-paste installation commands (`winget install Gyan.FFmpeg` or `brew install ffmpeg`).

---

## 3. The Annotated Recording Architecture

To ensure the demo looks professional, the recording script includes:
1. **`addInitScript` Resilience**: Injects the HUD styles and element before *any* page script runs. If the user flow navigates across full page loads or redirects, the HUD automatically re-initializes with the current step caption intact!
2. **Glassmorphic Floating Pill**: Renders at `z-index: 2147483647` with dark slate glassmorphism, crisp white typography, an accent step badge (`#0284c7`), and `pointer-events: none` so it never blocks clicks.
3. **Virtual Click Ripple**: Whenever an element is clicked, an expanding animated ripple appears at the exact click coordinates—solving the problem of headless recordings lacking a visible mouse cursor.
4. **Paced Dwell Times**: Injects 600–1000ms pauses between actions and a 2.5s hold on the final state before the GIF loops.

### The Recording Script: `scratch/recordings/record.js`

Author `scratch/recordings/record.js`:

```javascript
const { chromium } = require('playwright');
const path = require('path');
const fs = require('fs');

// --- Configuration ---
const CONFIG = {
  url: process.env.TARGET_URL || 'http://localhost:5173',
  outputDir: path.resolve('scratch/recordings'),
  viewport: { width: 1280, height: 720 },
  hudPosition: process.env.HUD_POS || 'bottom-left', // 'bottom-left', 'bottom-right', 'top-left', 'top-right', 'off'
  headless: process.env.HEADED !== 'true'
};

async function record() {
  if (!fs.existsSync(CONFIG.outputDir)) fs.mkdirSync(CONFIG.outputDir, { recursive: true });

  const browser = await chromium.launch({ headless: CONFIG.headless });
  const context = await browser.newContext({
    viewport: CONFIG.viewport,
    recordVideo: {
      dir: CONFIG.outputDir,
      size: CONFIG.viewport
    }
  });

  // Inject resilient HUD and Click Ripple styles on every page load/navigation
  await context.addInitScript(({ hudPos }) => {
    window.__demo_hud_config = window.__demo_hud_config || { step: 1, text: '', pos: hudPos };

    function ensureHUD() {
      if (hudPos === 'off') return;

      let style = document.getElementById('__demo_hud_styles__');
      if (!style) {
        style = document.createElement('style');
        style.id = '__demo_hud_styles__';
        
        // Dynamic positioning coordinates
        const posRules = {
          'bottom-left': 'bottom: 24px; left: 24px;',
          'bottom-right': 'bottom: 24px; right: 24px;',
          'top-left': 'top: 24px; left: 24px;',
          'top-right': 'top: 24px; right: 24px;'
        }[hudPos] || 'bottom: 24px; left: 24px;';

        style.textContent = `
          #__demo_annotation_hud__ {
            position: fixed;
            ${posRules}
            z-index: 2147483647;
            display: inline-flex;
            align-items: center;
            gap: 10px;
            padding: 9px 18px;
            background: rgba(15, 23, 42, 0.88);
            backdrop-filter: blur(12px);
            -webkit-backdrop-filter: blur(12px);
            color: #f8fafc;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            font-size: 13.5px;
            font-weight: 500;
            line-height: 1.4;
            border-radius: 9999px;
            border: 1px solid rgba(255, 255, 255, 0.18);
            box-shadow: 0 12px 30px -4px rgba(0, 0, 0, 0.4), 0 4px 6px -2px rgba(0, 0, 0, 0.2);
            transition: all 0.35s cubic-bezier(0.16, 1, 0.3, 1);
            pointer-events: none;
            user-select: none;
          }
          #__demo_annotation_hud__.hud-hidden {
            opacity: 0;
            transform: translateY(10px) scale(0.96);
          }
          .__demo_step_badge__ {
            background: #0284c7;
            color: #ffffff;
            font-weight: 700;
            font-size: 11px;
            text-transform: uppercase;
            letter-spacing: 0.05em;
            padding: 3px 8px;
            border-radius: 9999px;
            box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
          }
          .__demo_click_ripple__ {
            position: fixed;
            border-radius: 50%;
            pointer-events: none;
            z-index: 2147483646;
            border: 2px solid #38bdf8;
            background: rgba(56, 189, 248, 0.25);
            animation: __demo_ripple_anim__ 0.6s ease-out forwards;
          }
          @keyframes __demo_ripple_anim__ {
            0% { transform: translate(-50%, -50%) scale(0.2); opacity: 1; }
            100% { transform: translate(-50%, -50%) scale(2.8); opacity: 0; }
          }
        `;
        document.documentElement.appendChild(style);
      }

      let hud = document.getElementById('__demo_annotation_hud__');
      if (!hud) {
        hud = document.createElement('div');
        hud.id = '__demo_annotation_hud__';
        document.documentElement.appendChild(hud);
      }

      if (window.__demo_hud_config.text) {
        hud.innerHTML = `<span class="__demo_step_badge__">Step ${window.__demo_hud_config.step}</span> <span>${window.__demo_hud_config.text}</span>`;
        hud.classList.remove('hud-hidden');
      } else {
        hud.classList.add('hud-hidden');
      }
    }

    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', ensureHUD);
    } else {
      ensureHUD();
    }
  }, { hudPos: CONFIG.hudPosition });

  const page = await context.newPage();
  const pause = (ms) => new Promise(res => setTimeout(res, ms));

  // --- Helper: Update Annotation HUD ---
  async function setHUD(step, text) {
    console.log(`[Step ${step}] ${text}`);
    if (CONFIG.hudPosition === 'off') return;
    await page.evaluate(({ s, t }) => {
      window.__demo_hud_config = { step: s, text: t, pos: window.__demo_hud_config?.pos };
      const hud = document.getElementById('__demo_annotation_hud__');
      if (hud) {
        hud.innerHTML = `<span class="__demo_step_badge__">Step ${s}</span> <span>${t}</span>`;
        hud.classList.remove('hud-hidden');
      }
    }, { s: step, t: text });
    await pause(500); // Cognitive dwell time so viewers read the caption
  }

  // --- Helper: Click with Visual Ripple ---
  async function clickWithRipple(selector) {
    const element = await page.waitForSelector(selector, { state: 'visible', timeout: 8000 });
    const box = await element.boundingBox();
    if (box) {
      const clickX = box.x + box.width / 2;
      const clickY = box.y + box.height / 2;
      await page.evaluate(({ x, y }) => {
        const ripple = document.createElement('div');
        ripple.className = '__demo_click_ripple__';
        ripple.style.left = `${x}px`;
        ripple.style.top = `${y}px`;
        ripple.style.width = '24px';
        ripple.style.height = '24px';
        document.documentElement.appendChild(ripple);
        setTimeout(() => ripple.remove(), 700);
      }, { x: clickX, y: clickY });
    }
    await element.click();
    await pause(800); // Allow UI mutation or transition to settle
  }

  // --- Helper: Type realistically with character delay ---
  async function typeWithPacing(selector, text) {
    const input = await page.waitForSelector(selector, { state: 'visible', timeout: 8000 });
    await input.focus();
    await pause(200);
    await input.type(text, { delay: 65 });
    await pause(600);
  }

  // ==========================================================
  // --- EXECUTE CHOREOGRAPHED USER STORY FLOW ---
  // ==========================================================

  console.log('Starting recorded user story flow...');
  await page.goto(CONFIG.url);
  await pause(1200);

  // --- Step 1: Replace with your actual user story steps ---
  await setHUD(1, "Navigate to Dashboard");
  await pause(800);

  // Example Step 2:
  // await setHUD(2, "Open Filter Menu");
  // await clickWithRipple('[data-testid="filter-button"]');

  // Example Step 3:
  // await setHUD(3, "Search for 'Analytics'");
  // await typeWithPacing('[data-testid="search-input"]', 'Analytics');
  // await page.keyboard.press('Enter');

  // Example Step 4:
  // await setHUD(4, "Confirm Results Loaded");
  // await pause(1500);

  // --- Final Hold: Freeze on completed state before GIF loop resets ---
  console.log('Holding final state for GIF loop...');
  await pause(2500);

  // Closing context flushes the video file cleanly
  await context.close();
  const videoPath = await page.video().path();
  await browser.close();

  // Write video path to temporary reference file
  fs.writeFileSync(path.join(CONFIG.outputDir, 'latest_video.txt'), videoPath, 'utf8');
  console.log(`RECORDING_SUCCESS: ${videoPath}`);
}

record().catch(err => {
  console.error('[RECORD_ERROR]', err);
  process.exit(1);
});
```

---

## 4. Execute Recording & Verify Raw Capture

Execute the recording script:

**PowerShell:**
```powershell
node scratch/recordings/record.js
$videoFile = Get-Content "scratch/recordings/latest_video.txt" -Raw
Write-Host "Raw WebM captured at: $videoFile" -ForegroundColor Cyan
```

**Bash:**
```bash
node scratch/recordings/record.js
video_file=$(cat scratch/recordings/latest_video.txt)
echo "Raw WebM captured at: $video_file"
```

Verify that the `.webm` exists and is non-empty before proceeding to GIF conversion.

---

## 5. Two-Pass Color Palette GIF Optimization

Convert the raw `.webm` into a high-craft `.gif` using 2-pass color quantization.

### Filtergraph Details:
- `fps=15`: Perfectly smooth motion without redundant frame bloat.
- `scale=800:-1:flags=lanczos`: Scales width to `800px` maintaining native aspect ratio with high-quality lanczos downsampling.
- `palettegen=max_colors=128:reserve_transparent=0`: Analyzes color variance across the entire clip to generate an optimal custom palette.
- `paletteuse=dither=bayer:bayer_scale=3`: Applies ordered Bayer dithering to eliminate harsh banding while keeping file size small.

#### Option A: Using `ffmpeg` (Recommended)

**PowerShell:**
```powershell
$videoFile = (Get-Content "scratch/recordings/latest_video.txt" -Raw).Trim()
$outputGif = "assets/demos/feature-demo.gif"
$fps = 15
$width = 800

# Create destination folder
$destDir = Split-Path -Parent $outputGif
if ($destDir -and -not (Test-Path $destDir)) { New-Item -ItemType Directory -Path $destDir -Force }

# Two-pass conversion
$filter = "fps=$fps,scale=$width:-1:flags=lanczos,split[s0][s1];[s0]palettegen=max_colors=128:reserve_transparent=0[p];[s1][p]paletteuse=dither=bayer:bayer_scale=3"
ffmpeg -y -i "$videoFile" -vf $filter "$outputGif"

# Report size
$sizeMB = [math]::Round((Get-Item $outputGif).Length / 1MB, 2)
Write-Host "[SUCCESS] Generated Annotated GIF: $outputGif ($sizeMB MB)" -ForegroundColor Green
```

**Bash:**
```bash
video_file=$(cat scratch/recordings/latest_video.txt | tr -d '[:space:]')
output_gif="assets/demos/feature-demo.gif"
fps=15
width=800

mkdir -p "$(dirname "$output_gif")"

filter="fps=${fps},scale=${width}:-1:flags=lanczos,split[s0][s1];[s0]palettegen=max_colors=128:reserve_transparent=0[p];[s1][p]paletteuse=dither=bayer:bayer_scale=3"
ffmpeg -y -i "$video_file" -vf "$filter" "$output_gif"

size_mb=$(du -m "$output_gif" | cut -f1)
echo "[SUCCESS] Generated Annotated GIF: $output_gif (${size_mb} MB)"
```

#### Option B: Python Fallback (if `ffmpeg` is unavailable)
```python
# scratch/recordings/convert_to_gif.py
import sys, os
from moviepy.editor import VideoFileClip

input_path = sys.argv[1]
output_path = sys.argv[2]
clip = VideoFileClip(input_path).resize(width=800)
clip.write_gif(output_path, fps=15, program='imageio', opt='nq')
print(f"Saved GIF: {output_path}")
```

---

## 6. Size Validation & Visual Quality Gate

Verify the resulting GIF meets professional standards:
1. **File Size Check:**
   - **Target:** `< 5.0 MB` (optimal for instant GitHub README loading).
   - **If 5MB – 8MB:** Acceptable for long or intricate flows.
   - **If > 8MB:** Re-run optimization with `--fps 12` and `--width 720` to ensure smooth loading without hitting GitHub asset throttling.
2. **Loop Ergonomics:**
   - Confirm that the 2.5-second holding pause at the end prevents jarring, instant looping.
3. **HUD Readability:**
   - Confirm the HUD text does not overlap with critical UI buttons or inputs. If it does, re-record with `--hud top-right` or `--hud bottom-right`.

---

## 7. Documentation & README Integration

1. **Promote the Asset:**
   Move the finalized GIF to its permanent location (e.g. `assets/demos/<flow-slug>.gif` or `docs/demos/<flow-slug>.gif`).

2. **Generate Markdown Embed Code:**
   Display the ready-to-use Markdown snippet:

   ```markdown
   ### User Story Walkthrough: <Story Title>

   ![<Story Title> Demo](assets/demos/<flow-slug>.gif)
   ```

3. **Prompt User to Insert into `README.md`:**
   Ask the user:
   > *"The annotated demo GIF has been generated at `assets/demos/<flow-slug>.gif` (Size: X.XX MB). Would you like me to insert this into your `README.md` now?"*

4. **Preserve Recording Script for Re-runs:**
   Save `scratch/recordings/record.js` as `tests/demos/<flow-slug>.demo.js`.
   Whenever future development phases update this UI, running `node tests/demos/<flow-slug>.demo.js` will immediately produce a freshly updated demo GIF.

5. **Clean Up Scratch Files:**
   Remove temporary video clips and text files from `scratch/recordings/` to keep the workspace clean.

</process>
