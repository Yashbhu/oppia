# Viewing server and console logs (dev, tests, Docker)

This document explains how to view backend (server) and frontend (browser/console) logs when
working with Oppia in different environments: local Python dev server, backend/frontend unit
tests, Puppeteer e2e/acceptance tests, and Docker-based runs.

Where to add this doc
- Repo: `docs/logging.md` (this file)
- Alternatively: add to the project wiki if you prefer web-hosted docs.

Quick principles
- Set explicit log levels when debugging (e.g. `INFO` or `DEBUG`).
- For Python tests, prefer `--log-cli-level=INFO` so log messages are shown during test runs.
- Forward browser console messages to stdout in your test harness (Puppeteer `page.on('console', ...)`).
- For Docker, use `docker logs -f` or `docker-compose logs -f` to stream logs.

1) Local Python dev server

Start the dev server (from the repo root):

```bash
python -m scripts.start
```

Notes:
- The script will install third-party libraries (unless you pass `--skip-install`) and run the local
  GAE dev server and the frontend build in watch mode.
- Server logs (backend) print to the terminal where you started `scripts.start`.
- Webpack/ng build output (frontend build logs) prints to the same terminal, and browser console
  logs are visible in the browser devtools Console.

Useful flags:
- Don't open a browser automatically:

```bash
python -m scripts.start --no_browser
```

- Run with source maps for easier stack traces:

```bash
python -m scripts.start --source_maps
```

2) Backend unit tests (Python)

Run an individual test or test target with the Oppia test runner:

```bash
python -m scripts.run_backend_tests --test-target core.controllers.android
```

If you prefer `pytest` directly, show logs and stdout:

```bash
pytest path/to/test_file.py -s --log-cli-level=INFO
```

Flags explained:
- `-s` shows stdout and `print()` output.
- `--log-cli-level=INFO` prints Python `logging` messages emitted during tests.

3) Frontend unit tests (Jest / Karma)

Root `package.json` may not contain test scripts; check `assets/` for frontend tooling. Typical commands:

```bash
# from the repo root
cd assets/
# run Jest tests (example; depends on repo scripts)
yarn test --testPathPattern=some.test.ts --runInBand --verbose
```

Notes:
- `--runInBand` runs tests serially, which can make logs easier to read.
- If a test runner silences console logs on success, run a single test (or use verbose flags) so you can
  see console output on failures.

4) Puppeteer / e2e / acceptance tests

Backend logs:
- If you start the backend server separately, watch the terminal where the server runs.
- If test harness starts the server, ensure the harness logs are visible.

Browser/console logs (from Puppeteer):
- Forward browser logs to stdout by adding a listener in the test setup. Example:

```js
// inside Puppeteer tests (Node)
page.on('console', msg => {
  console.log(`[PUPPETEER] ${msg.type().toUpperCase()}: ${msg.text()}`);
});
```

This will ensure console messages from the page (error, warning, log) appear in test output.

5) Docker-based runs

Find running containers or services:

```bash
docker ps --format 'table {{.ID}}\t{{.Names}}\t{{.Image}}'
```

Stream logs for a container:

```bash
docker logs -f <container_name_or_id>
```

If you use `docker-compose`:

```bash
docker-compose -f <compose-file.yml> logs -f <service>
# or all services
docker-compose -f <compose-file.yml> logs -f
```

Tips:
- If you have many containers, add `--tail=N` to only show the last N lines.
- Use `grep` to filter for particular log lines / services.

6) Lighthouse and other test types

Treat these like e2e runs: ensure the backend server logs are visible and capture browser console via the test harness or DevTools protocol.

7) Troubleshooting checklist

- I don't see backend logs: confirm you started the server in the foreground and that the process isn't
  running in the background or inside a container you are not attached to.
- I don't see frontend console logs in tests: ensure the test harness attaches a `page.on('console', ...)`
  handler or that the test runner isn't suppressing logs for passing tests.
- Docker logs empty: confirm the container is running (`docker ps`) and you're tailing the correct container name.

8) Suggested best-practices (for the repo)

- Add `page.on('console', ...)` to Puppeteer test helpers to always forward browser logs to stdout.
- When debugging CI failures, re-run the test with increased log levels: in Python use
  `--log-cli-level=DEBUG`, in frontend add verbose flags to Jest/Karma.
- Add a short `docs/logging.md` (this file) with examples for both Python and Docker workflows.

If you want, I can:
- Create this file in the repo (I will create a branch, commit the file, and push it) — I can do that now.
- Or open a PR draft and include reviewers/labels as you specify.

