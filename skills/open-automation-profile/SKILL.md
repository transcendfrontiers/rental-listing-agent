---
name: open-automation-profile
description: Open Google Chrome into the dedicated automation profile the Playwright MCP server drives for the listing skills (separate from the user's everyday Chrome profile), e.g. to log into Vrbo/Zillow/TurboTenant/Airbnb once, clear a bot check, or watch a run. Use when the user asks to open/launch the automation browser or profile.
---

# open-automation-profile

Launches Chrome into **the same profile storage the Playwright MCP server automates**, so the user can log in once (sessions persist for later runs), clear a "Press & Hold" bot check, or watch the agent work.

Why a separate profile: Chrome won't let two processes share one profile directory. Giving the agent its own `userDataDir` keeps it from colliding with — or reading — the user's everyday browser, and its logins persist between runs.

## Steps (macOS)

1. Find the Playwright config path in `host-profile.yaml` → `browser.playwright_config` (default `~/.claude/playwright-mcp-config.json`). Read it and take `browser.userDataDir` and the `--profile-directory=` value from `browser.launchOptions.args` (default `Default`). Re-read it every time; don't trust cached values.
2. Run:

   ```bash
   open -na "Google Chrome" --args \
     --user-data-dir="<userDataDir>" \
     --profile-directory="<profile-directory>"
   ```

   `-na` opens a new Chrome instance even if Chrome is already running on another profile. Don't pass a URL unless the user asked for one.

3. Report that the automation profile is open. A visible Chrome window is confirmation enough.

On Linux, run `google-chrome --user-data-dir=... --profile-directory=...` instead. On Windows, run `chrome.exe` with the same flags.

If an agent session is already driving this profile, this just opens another window into it. That's fine if the user wants to watch or step in.
