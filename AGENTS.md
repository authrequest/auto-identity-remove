# AGENTS.md — auto-identity-remove

High-signal context for agents working in this repo.

## Running the tool

```bash
# First-time setup (creates config.json)
node setup.js

# Manual run
./run.sh
# or
node watcher.js

# Dry-run mode (fills forms but doesn't submit)
node watcher.js --dry-run

# Run tests
npm test
```

## Key facts

- **Node 18+ required** — check `engines` in `package.json`
- **macOS/Linux only** — Windows not supported
- **Playwright browsers required** — run `npx playwright install chromium` before first run
- **Config is gitignored** — `config.json` and `state.json` must not be committed
- **Browser profile persists** — stored at `~/.config/auto-identity-remove` by default; reused across runs for saved logins

## Architecture

```
watcher.js        → Main orchestrator (entry point)
brokers.js        → 30+ explicit broker definitions (requires config.json at load)
generic-runner.js → Handles ~470 brokers from data/markup-parsed.json + data/badbool-extra.json
lib/
  config.js       → Config/state loading, dry-run guard, 90-day re-check logic
  broker-runner.js → Per-broker processing, email opt-outs
  captcha.js      → CapSolver API integration for reCAPTCHA solving
  forms.js        → Form filling, listing URL discovery
  logger.js       → Results accumulator, summary builder
  notify.js       → iMessage (macOS) / notify-send (Linux)
  platform.js     → Platform detection (not yet used)
```

## Important quirks

- **brokers.js requires config.json** — it imports config at module load time. If config.json doesn't exist, `require('./brokers')` will crash. Always run `node setup.js` first.
- **Dry-run is read-only** — it navigates to URLs and dismisses banners but stops before clicking "Do Not Sell" buttons or submitting forms. The `--dry-run` flag also gates state.json writes.
- **90-day re-check window** — brokers within this window are skipped. Logic lives in `lib/config.js` (`RECHECK_DAYS`, `lastOptOutDaysAgo`).
- **Browser detection evasion** — watcher.js passes `--disable-blink-features=AutomationControlled` and `ignoreDefaultArgs: ['--enable-automation']` to Playwright.
- **Generic broker deduplication** — generic-runner skips any hostnames already in explicit brokers.js.
- **Platform-specific scheduling** — setup.js creates launchd (macOS) or systemd timer (Linux) for monthly runs.

## Adding new brokers

Edit `brokers.js`. Available methods:

- `search-form` — search for person, extract listing URL, then opt-out
- `direct-form` — go straight to opt-out URL
- `email` — send removal email (uses `emailTo` field)
- `manual` — too complex, added to manual list

Form field placeholders: `F`, `L`, `N`, `E`, `ST`, `C`, `Z` map to firstName, lastName, fullName, email, state, city, zip from config.json.

## Data files

- `data/markup-parsed.json` — The Markup's 494 broker URLs
- `data/badbool-extra.json` — 27 additional community-sourced sites

Both are loaded by generic-runner.js and deduplicated against explicit brokers.js.

## Test notes

- Tests use Node's built-in `node:test`
- Tests mutate the live `state.json` in-memory object and restore after
- Run `npm test` to execute all tests
