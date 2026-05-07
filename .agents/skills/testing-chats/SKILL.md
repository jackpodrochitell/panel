---
name: testing-chats
description: End-to-end runtime testing for the Chats page of the FunPay-killer panel — how to start the backend + Vite dev server, sign in, add a FunPay account via golden_key, and exercise the chat list / pane / right sidebar. Use when verifying any change to `frontend/src/pages/Chats.tsx`, `frontend/src/components/RightSidebar.tsx`, or `frontend/src/index.css`.
---

# Testing the Chats page

## Repo layout

Source code lives **inside** `panel-devin-*-unpack-audit-fix.zip` at the repo root. The unpacked folder is **not** tracked. Workflow:

1. Unzip the archive (`unzip -oq panel-devin-*.zip`).
2. Edit files in `panel-devin-*/`.
3. Re-zip with the same filename (excluding `.venv`, `data/`, `funpay_killer.egg-info/`, `plugins-installed/`, `node_modules`, `dist`, `app/static/`, `__pycache__`, `*.pyc`, `.pytest_cache`, `.env`).
4. Commit only the updated `.zip`.

GitHub Actions workflow lives inside the zip, so **CI does not run on PRs that only modify the archive.** Don't wait for CI — verify locally.

## Local startup

Run the backend and the frontend in two separate shells, both as foreground processes so HMR / log output is visible:

```bash
base=$(ls -d panel-devin-*/ 2>/dev/null | head -n 1)

# Shell 1: backend (FastAPI)
cd "$base"
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
python run.py --dev      # binds 127.0.0.1:8000

# Shell 2: frontend (Vite)
cd "$base/frontend"
npm install
npm run dev               # http://localhost:5173
```

The frontend origin must be allowed by `FPK_ALLOWED_ORIGIN` in `.env`. The default `.env.example` uses `http://localhost:5173` — match that exactly (do **not** use `127.0.0.1:5173`, since Vite binds to `localhost` by default and CORS treats them as different origins).

## First-run flow

1. Visit `http://localhost:5173/setup`. Create the admin user (any password).
2. Visit `/accounts`, click **+ Add account**. Paste a FunPay `golden_key` (a 32-character session cookie). Label it.
3. To test account-switching UX, add **a second account using the same `golden_key`** — duplicates are accepted, and this is the cheapest way to make the chip-switching behaviour observable.
4. Visit `/chats`. The chat list, chat pane, and right "Currently viewing" sidebar should all render.

## What to look at when reviewing Chats CSS/animation changes

- **Sidebar toggle (top-right of the chat pane).** Click it 5+ times rapidly. The right `.chat-grid__sidebar` column should slide in/out via `grid-template-columns` + a `translateX(20px)` transform + opacity fade. The toggle icon's vertical divider line should animate `translateX` in sync with the panel.
- **Scrollbars at the chat-grid edges.** No horizontal scrollbar at the bottom, no vertical scrollbar at the right of the grid container during the 280ms animation window. (`.chat-grid` has `overflow: hidden` to enforce this.)
- **Chat-pane text colours during toggle.** The chat header (title, online dot, `Chat #...` label) and the message bubbles must stay in their steady-state colours. `.chat-grid__sidebar` has `will-change: transform, opacity` so it gets its own compositor layer and doesn't repaint adjacent columns.
- **Account chips at the top.** Active = `.chip` (rounded-full pill, shadow-neu-inset). Inactive = `.btn-ghost` (transparent). The buttons override the default `transition-all` from `.btn` with `transition-none`, so the swap is instantaneous — no morphing of border-radius / padding / background should ever be visible mid-frame.
- **Outgoing bubbles ('me' side).** With the sidebar closed (chat-pane at its widest — worst case for overflow), bubbles must leave a visible right-margin gap from the pane edge. The messages container uses `pr-3` (12 px); the per-bubble `max-w-[80%]` enforces width, not the bubble class itself.

## Test-mode quick checklist

- [ ] Toggle sidebar 5+ times rapidly. Watch for scrollbar flicker, header colour flicker, divider sync.
- [ ] With sidebar closed, zoom on the right edge of the chat-pane. Verify outgoing bubbles + image attachments leave a gap.
- [ ] With 2+ accounts configured, click between chips 3+ times. Verify instant shape swap.
- [ ] (Optional) Toggle dark/light theme via the bottom-left sun/moon button. The Chats fixes should be theme-agnostic (CSS vars only).
- [ ] (Optional) Set `prefers-reduced-motion: reduce` in DevTools rendering panel. The toggle icon animation should be disabled but the panel still resizes (acceptable trade-off — only the icon transitions are gated by the media query).

## Recording / annotations

For visual changes, record a session and use `annotate_recording`:
- `setup` for navigation steps (e.g. "Adding a 2nd account").
- `test_start` for each named test ("It should not flash scrollbars on the chat-grid during sidebar toggle").
- `assertion` with `passed`/`failed`/`untested` after each verification ("Sidebar closed; no scrollbar at chat-grid edges").

Record the verification run in **light mode** by default — the user originally reported the issues in dark mode but the underlying fixes are CSS-var-agnostic, so verifying in either theme is fine. Call this out in the test report if you only verify one.

## Devin Secrets needed

- `FUNPAY_GOLDEN_KEY` (per-user secret, `should_save=true, save_scope='user'`) — a 32-character FunPay session cookie. Pass via the **+ Add account** UI form, never via the URL or env. Don't hard-code a real key into a SKILL or a test plan; ask the user via `request_secret` if a saved one is not in the environment.

## Known issues / workarounds

- **`browser_console` may report "Chrome is not in foreground"** even when Chrome is the active X11 window. Fall back to (a) `curl http://localhost:5173/src/index.css` to verify the bundled CSS contains your changes, or (b) opening DevTools (`F12`) and running JS in the in-browser Console — but note that opening DevTools collapses the chat-grid layout, so close it before taking screenshots.
- **Maximizing the Chrome window via `xdotool key super+Up`** tiles the window to half-screen on this Plasma window manager. Use `wmctrl -i -r <window-id> -e 0,0,0,1600,1170` to explicitly resize to the full X-screen size (1600x1200) instead.
- **CI never runs** on this repo for PR-only zip changes — the workflow file is inside the zip. Don't wait on `git_pr_checks`.
