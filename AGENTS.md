# AGENTS.md — Cryndel Launcher

Electron 27 Minecraft launcher. Two codebases live here:

- `electron/` (main process) + `renderer/` (UI) — **the active codebase**.
- `src/` — old/deprecated. Do not edit, do not "fix" it.

## Commands

```bash
npm install            # required after a fresh checkout; node_modules may be absent
npm start              # run the launcher
npm test               # jest: 14 suites, 47 tests (roots tests/unit)
npx eslint --config eslintrc.json electron renderer --ext .js
```

## Headless end-to-end smoke test

There is no X server in the dev container, so install xvfb once and run:

```bash
sudo apt-get update && sudo apt-get install -y xvfb
CRYNDEL_HOME=/tmp/cryndel-e2e timeout 30 xvfb-run -a node_modules/.bin/electron . \
  --no-sandbox --disable-gpu
# exit 124 = app still alive (good); exit 0 = app quit early (bug)
```

`CRYNDEL_HOME` redirects all launcher state to a throwaway directory.
`--no-sandbox` is needed because the SUID sandbox helper is not configured here.
The dbus `bus.cc` errors in the log are harmless container noise.

## Gotchas that have caused real bugs

- **`browser-window-created` fires synchronously inside `new BrowserWindow()`.**
  Any handler comparing `win === panes.win` fails for the launcher's own window,
  because the assignment has not happened yet. Use `panes.creating` (set by
  `PaneSmith.create()`) to recognise it. Getting this wrong destroyed the main
  window right after boot and the app exited immediately after "kernel ready".
  See `electron/popup-guard.js` and `tests/unit/window-boot.test.js`.
- **webview `partition` must be set before `src`.** Electron only honours it on
  first navigation, so setting it afterwards silently drops the page into the
  default session and logins/cookies are not persisted.
- **Hotkeys must use `e.code`, not `e.key`.** On a Russian keyboard layout
  `e.key` is a Cyrillic letter, so `KeyJ` never matched.
- **Downloads must be attached to every session**, including `persist:cryndel`;
  `session.defaultSession` alone is not the one the webviews use.
- Never clear `persist:cryndel` storage on a settings change — it holds the
  user's logins.

## Feature-flag defaults

3D skins (`showSkin3d`) and Discord RPC are intentionally **off by default**.
Keep the code, but do not re-enable them by default.

## Rules

- Do not remove or trim existing features.
- Keep the UI adaptive; check the `@media` breakpoints in `renderer/css/`.
