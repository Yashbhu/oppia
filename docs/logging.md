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
 
---

Full Oppia-ready logging checklist
Below are explicit, actionable instructions covering the 18 items requested: 9 for the Python (local) workflows and 9 for the Docker workflows. Each subsection explains which process/container prints the logs and gives copy-paste `zsh` commands to view them.

PART A — Python workflows (local, non-Docker)

A1) Dev server — backend logs
- What prints the logs: the Python process started by `python -m scripts.start` (this launches the dev GAE dev_appserver and backend services). Backend log lines (request handlers, logging.info/error) are printed to the same terminal running `scripts.start`.
- How to run & view:

```bash
# Start the dev server (from repo root)
python -m scripts.start

# View logs: watch the terminal where you ran the command — backend logs appear inline.
```

Notes: If you started the server in the background, bring it to the foreground or inspect the process output (e.g. by running it in a terminal multiplexer or redirecting output to a file).

A2) Dev server — frontend logs (webpack/ng build and browser console)
- What prints the logs:
  - Webpack / frontend build (compiler) logs are produced by the build process that `scripts.start` launches (managed webpack/ng build) — they are printed to the same terminal as `scripts.start`.
  - Browser console logs appear in the browser devtools Console when you open `http://localhost:8181`.
- How to run & view:

```bash
# Start dev server (same as above) — webpack/ng build output appears in the same terminal
python -m scripts.start

# Open browser devtools (Cmd+Opt+I on macOS) and view Console for frontend runtime logs.
```

Notes: If you only want the build logs, run the build step directly (for debug) or watch the `scripts.start` terminal where `managed_webpack_compiler` prints compilation messages.

A3) Backend unit tests (Python)
- What prints the logs: the test runner process (the `python -m scripts.run_backend_tests` wrapper or `pytest`) prints logging output to stdout/stderr when configured.
- How to run & view:

```bash
# Run an Oppia backend test target via the project helper
python -m scripts.run_backend_tests --test-target core.controllers.android

# Or use pytest directly to see logging and stdout
pytest core/controllers/android_test.py -s --log-cli-level=INFO
```

Notes: Use `-s` to show stdout/print and `--log-cli-level=INFO` (or DEBUG) to show Python logging emitted during tests.

A4) Frontend unit tests (local, Jest/Karma)
- What prints the logs: the frontend test runner (Jest / Karma) prints logs to the terminal where you run the test command (usually from `assets/`).
- How to run & view:

```bash
cd assets/
# Example: run Jest tests with verbose logging
yarn test --testPathPattern=some.test.ts --runInBand --verbose

# Or run a specific Karma/Jasmine test harness if the repo uses Karma (check assets/ for scripts)
```

Notes: `--runInBand` makes console logs easier to follow locally.

A5) Backend logs during e2e tests (Puppeteer-driven acceptance/e2e)
- What prints the logs: when you run Puppeteer e2e/acceptance tests locally via `python -m scripts.run_backend_tests --test-target core/tests/puppeteer-acceptance-tests/...`, two cases exist:
  - If you start the backend server manually (via `python -m scripts.start`), backend logs appear in the server terminal.
  - If the test runner starts the server internally, server and harness logs appear in the terminal that runs the `python -m scripts.run_backend_tests` command.
- How to run & view:

```bash
# Run a single Puppeteer acceptance spec via the runner
python -m scripts.run_backend_tests --test-target \
  core/tests/puppeteer-acceptance-tests/specs/logged-in-user/view-subtopic-study-guides.spec.ts

# If you started the server manually, tail its terminal; otherwise inspect the terminal running the above command.
```

Notes: The Puppeteer harness in this repo includes `core/tests/puppeteer-acceptance-tests/utilities/common/console-reporter.ts`, which forwards browser console messages into the test output (see section A6).

A6) Frontend logs during e2e tests
- What prints the logs: browser console messages are captured by the test harness (the `ConsoleReporter` in the repo) and are included in the test runner output. This means frontend console logs will appear in the terminal running the acceptance test command.
- How to verify / view:

```bash
# Run the acceptance test (same as A5) — browser console messages are forwarded to the test output
python -m scripts.run_backend_tests --test-target core/tests/puppeteer-acceptance-tests/specs/your.spec.ts

# You can also inspect console-reporter.ts to change filtering or make messages more verbose
sed -n '1,240p' core/tests/puppeteer-acceptance-tests/utilities/common/console-reporter.ts
```

Notes: The harness ignores some expected noisy messages; see `console-reporter.ts` for the ignore list and how errors are reported.

A7) Backend logs during acceptance tests
- What prints the logs: same as A5 — acceptance tests (the full suite) either run against a manually started server (use `scripts.start`) or a server launched by the runner. Backend logs will appear in whichever terminal is running the server or test harness.
- How to run & view:

```bash
# Run full acceptance suite via helper (this can take time)
python -m scripts.run_backend_tests --test-target core/tests/puppeteer-acceptance-tests

# Tail server terminal if started separately
```

Notes: For debugging, start the server manually (`python -m scripts.start`) in one terminal and run the tests in another terminal so you can see server logs and test output side-by-side.

A8) Frontend logs during acceptance tests
- What prints the logs: frontend console output is forwarded by the Puppeteer test harness (ConsoleReporter) into the test runner output. If you started the browser manually, use browser devtools.
- How to run & view: same commands as A6/A7.

A9) Backend + frontend logs during Lighthouse runs (local / Python)
- What prints the logs:
  - Lighthouse runs in a Node/Puppeteer context and the test harness invokes the browser; backend logs come from the server terminal (where `scripts.start` was run or where the test harness launched the server).
  - Frontend console logs can be captured in the Lighthouse/Puppeteer script by attaching `page.on('console', ...)` (the repo uses `puppeteer-login-script.js` and `.lighthouserc-base.js` to configure runs).
- How to run & view:

```bash
# Example: run a Lighthouse script that uses Puppeteer (repo uses .lighthouserc-base.js)
node tools/lighthouse-runner.js  # or follow repo's lighthouse helper (check .lighthouserc-base.js)

# Watch the server terminal (backend logs) and the Node process output (Lighthouse/Puppeteer logs + forwarded console messages)
```

PART B — Docker workflows (service/container-specific)

Notes about Oppia Docker layout: the project's `docker-compose.yml` defines the primary services and container names used during development. Important service/container names (as defined in the repo):

- `oppia-dev-server` — the main backend dev server container (service: `dev-server`). This is where Oppia's Python backend runs in Docker.
- `oppia-webpack-compiler` — webpack compiler for the frontend; shows build and compilation logs.
- `oppia-angular-build` — used for frontend build stages (may print build/test logs depending on which service runs tests).
- `oppia-firebase-emulator` — Firebase emulator container (console output for firebase emulator).
- `oppia-cloud-datastore` — datastore emulator container.

B1) Dev server — backend logs (Docker)
- Which container prints logs: `oppia-dev-server` prints backend logs (app handlers, logging.*) to its stdout.
- How to view:

```bash
# Start services (if not already running)
docker compose up -d dev-server

# Follow backend logs
docker logs -f oppia-dev-server

# If you want to run the dev server attached in the foreground instead
docker compose up dev-server
```

Notes: When the service runs other helper containers (datastore, redis, elasticsearch), their logs are in their containers (see `docker ps`).

B2) Dev server — frontend logs (Docker)
- Which containers print logs:
  - Build/compiler logs: `oppia-webpack-compiler` and `oppia-angular-build` print webpack/ng build messages.
  - Browser console logs are not printed by these containers — they are emitted by the browser runtime. To capture browser console logs in tests, use Puppeteer harness which forwards them into test output.
- How to view build logs:

```bash
docker logs -f oppia-webpack-compiler
docker logs -f oppia-angular-build
```

Notes: In Docker-based development, you typically run the frontend compilation service and watch its logs to see rebuilds and bundler warnings/errors.

B3) Backend unit tests (Docker)
- Which container prints logs: backend unit tests run in the backend context — typically executed inside the `oppia-dev-server` container (or with a one-off run against that service). The pytest/logging output from tests will be printed to the terminal that invoked the test command.
- How to run & view:

```bash
# Run backend tests inside a disposable container (example)
docker compose run --rm dev-server python -m scripts.run_backend_tests --test-target core.controllers.android

# Follow logs from the container while it runs
docker logs -f $(docker ps -q -f name=oppia-dev-server)
```

Notes: The `docker compose run` command will stream the test output to your current terminal by default. If you run tests from inside a long-running `oppia-dev-server`, use `docker exec -it oppia-dev-server bash` and run the tests there.

B4) Frontend unit tests (Docker)
- Which container prints logs: frontend unit tests (Jest/Karma) are executed either in `oppia-angular-build` or `oppia-webpack-compiler` depending on your chosen workflow. Running the test command in a container will print the test logs to the invoking terminal.
- How to run & view:

```bash
# Example: run Jest in a disposable frontend container
docker compose run --rm webpack-compiler bash -lc "cd assets && yarn test --testPathPattern=some.test.ts --runInBand --verbose"

# Or check the compiler build logs for compile-time errors
docker logs -f oppia-webpack-compiler
```

B5) Backend logs during e2e tests (Docker)
- Which container prints logs: the backend logs appear in `oppia-dev-server` (if server runs in Docker). If your e2e harness runs in a container, that harness's stdout will also include test logs.
- How to run & view:

```bash
# Start services needed for e2e
docker compose up -d dev-server webpack-compiler firebase datastore redis elasticsearch

# Run the e2e acceptance test inside the backend container
docker compose run --rm dev-server python -m scripts.run_backend_tests --test-target \
  core/tests/puppeteer-acceptance-tests/specs/your.spec.ts

# Tail server logs concurrently
docker logs -f oppia-dev-server
```

Notes: When tests execute inside containers, the test runner output (including forwarded browser console lines) will be streamed to your terminal by the `docker compose run` invocation.

B6) Frontend logs during e2e tests (Docker)
- Which container prints logs: browser console logs are captured by the Puppeteer harness and will appear in the test runner's stdout (the container running the harness). Build/compiler warnings appear in `oppia-webpack-compiler`.
- How to run & view:

```bash
# Run the acceptance spec via the backend/test container (same as B5)
docker compose run --rm dev-server python -m scripts.run_backend_tests --test-target core/tests/puppeteer-acceptance-tests/specs/your.spec.ts

# Also optionally tail compiler logs
docker logs -f oppia-webpack-compiler
```

Notes: Containerized Puppeteer harnesses often forward browser console messages to stdout via the `ConsoleReporter` helper; check that file if messages are missing.

B7) Backend logs during acceptance tests (Docker)
- Which container prints logs: `oppia-dev-server`. Acceptance tests are the broader Puppeteer suite; backend logs appear in the dev-server container or in the test harness container if it starts the server.
- How to run & view:

```bash
docker compose up -d dev-server webpack-compiler firebase datastore redis elasticsearch
docker compose run --rm dev-server python -m scripts.run_backend_tests --test-target core/tests/puppeteer-acceptance-tests

# Tail server
docker logs -f oppia-dev-server
```

B8) Frontend logs during acceptance tests (Docker)
- Which container prints logs: forwarded into the test runner container's stdout (usually the `dev-server` run invocation). Compiler logs are in `oppia-webpack-compiler`.
- How to view:

```bash
docker compose run --rm dev-server python -m scripts.run_backend_tests --test-target core/tests/puppeteer-acceptance-tests/specs/your.spec.ts
docker logs -f oppia-webpack-compiler
```

B9) Backend + frontend logs during Lighthouse tests (Docker)
- Which container prints logs:
  - Backend server logs: `oppia-dev-server`.
  - Lighthouse / Puppeteer logs: the container or host process running the Lighthouse script (if you run Lighthouse inside Docker, it will be the container you launched — otherwise the host Node process prints them).
- How to run & view:

```bash
# Start required services
docker compose up -d dev-server webpack-compiler firebase datastore redis elasticsearch

# Run Lighthouse script inside a Node-capable container (example)
docker compose run --rm webpack-compiler node tools/lighthouse-runner.js

# Tail backend server logs
docker logs -f oppia-dev-server

# Tail compiler logs (optional)
docker logs -f oppia-webpack-compiler
```

Final notes & troubleshooting
- If logs are missing or truncated in Docker, check that the container/service name matches the names in `docker-compose.yml` (e.g. `oppia-dev-server`, `oppia-webpack-compiler`, `oppia-angular-build`).
- Use `docker ps` to list running containers and confirm names:

```bash
docker ps --format 'table {{.ID}}\t{{.Names}}\t{{.Image}}'
```

- Use `docker compose logs -f <service>` to show combined logs with service prefixes, which can be helpful when multiple services print interleaved logs.
- For acceptance and e2e tests, prefer starting the server manually (`python -m scripts.start` or `docker compose up dev-server`) and running tests in another terminal to keep server logs and test output separate and easier to compare.

If you want, I can also open a PR description and add reviewers/labels — tell me the PR title, description and any reviewers to add and I'll create the PR draft for you.

