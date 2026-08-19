# Capture fixture

`BOARD_FIXTURE=$(pwd)/tests/fixture bun run src/server.ts` boots an inert
board: config from this dir, canned data endpoints, no tokens, no timers,
no rt relay. Port 7941 (never the live board's 7930).

- `config.json` — committed fixture config (this dir).
- `data.json`, `discussions.json`, `review-report.md`, `meta.json` —
  committed by the baseline task; see tests/capture.ts.

`data.json` holds a real (private-repo) snapshot; timestamps are pinned and
every capture run freezes the browser clock to `meta.json`'s `now`.
