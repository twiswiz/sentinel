# Sentinel — Build Spec for Claude Code

Oct 14, 2026 · @Deepak

This spec is the single source of truth for building Sentinel: a small FastAPI + React app with an ML `/predict` endpoint, and a layered automated test suite that runs in GitHub Actions. Build it phase by phase in the order of the build phases section.

## How to use this spec

Export this doc as Markdown, put it in an empty repo as `docs/SPEC.md`, add the `CLAUDE.md` below at the repo root, then ask Claude Code for one phase at a time.

1. Create an empty GitHub repo (e.g. `sentinel`), clone it, and open it in your editor.
2. Export this doc as Markdown and save it as `docs/SPEC.md`.
3. Create `CLAUDE.md` at the repo root with the content below. Claude Code reads it automatically at the start of every session.
4. Start Claude Code in the repo and prompt: **"Read docs/SPEC.md and build Phase 1. Stop when its acceptance checks pass."**
5. Review the diff, push, and check that GitHub Actions is green. Then prompt for Phase 2, and so on.

One phase per session keeps each diff reviewable. If a phase goes wrong, you can reset to the last commit without losing earlier phases.

**`CLAUDE.md`**

```markdown
# Sentinel: AI Service + Automated Testing Framework

The full spec is in docs/SPEC.md. Read it before any work. It is the source of truth.

## How to work
- Build ONE phase at a time (see "Build phases" in the spec). Never start the next phase unprompted.
- At the end of a phase, run every acceptance check listed for it, show the output,
  then commit with message "phase N: <short summary>" and stop with a short summary.
- Follow the API contract, field names, status codes and UI copy EXACTLY.
  Tests, the Postman collection and the frontend all depend on them.
- Never delete, skip, xfail or loosen a test to make it pass. Fix the code.
  If the spec itself looks wrong, stop and ask.
- No features beyond the spec.

## Commands
- Backend tests:  pytest -m unit | integration | regression | api | performance
- Frontend tests: cd frontend && npm test
- E2E:            docker compose up -d --build --wait && cd frontend && npx playwright test
- Lint:           ruff check . && ruff format --check . ; cd frontend && npm run lint
```

## Working rules for Claude Code

These rules exist because the project's value is the tests: a green suite that was made green by weakening it is worth nothing.

1. **One phase at a time.** Build only the phase you were asked for. End it by running its acceptance checks, showing the output, committing as `phase N: <summary>`, and stopping.
2. **The contract is exact.** Endpoint paths, JSON field names, status codes, error messages, form labels, button text and `data-testid` values are written in this spec on purpose. Use them verbatim.
3. **Tests are not negotiable.** Never delete, skip, `xfail`, loosen a tolerance or lower a coverage gate to get green. Fix the code. If a test and the spec disagree, the spec wins; if the spec seems wrong, stop and ask.
4. **Write tests alongside code**, in the same phase, against the test requirements section. A phase isn't done if its tests are missing.
5. **Pin dependencies.** Use the latest stable versions at build time within the major versions listed, pinned exactly in `requirements*.txt` and `package-lock.json`.
6. **Keep it small.** No features, endpoints, pages or libraries beyond this spec. No UI component library.
7. **No real secrets.** Only the dev defaults in `.env.example`. Never commit `.env`.
8. **Keep the README current.** Each phase adds its "how to run" commands to `README.md`.
9. **Determinism.** Every random process takes seed `42`. Tests never depend on wall-clock time except the explicit latency tests, and never on test execution order.

## What we're building

Sentinel is a mini "loan approval" AI service with a web UI, wrapped in an automated test framework. The app is deliberately tiny; the test suite and CI are the product.

- **Web app (React):** sign in / sign up, a Predict page (age + income → label + probability), a History page of your past predictions, and a Test Dashboard showing the latest test-run results and service stats.
- **API (FastAPI):** health, users, login, predict, prediction history, prediction stats, latest test report.
- **Database (MySQL 8):** users and a log of every prediction.
- **Model (scikit-learn):** logistic regression on two features, trained on synthetic data, shipped as a versioned file with a checksum.
- **Tests:** backend unit, integration, API (black-box), regression and performance; frontend component tests; Playwright end-to-end tests; a Postman collection run by Newman; a Locust load test.
- **CI:** GitHub Actions runs lint → backend + frontend → e2e on every push, performance nightly, with coverage gates.

**Out of scope:** cloud deployment, real user data, model accuracy tuning, password reset, email sending, admin roles, refresh tokens, a mobile app, any UI component library.

## Tech stack and versions

Use the latest stable release within each listed major version and pin it exactly.

| Area | Choice | Version constraint |
| --- | --- | --- |
| Backend runtime | Python | 3.12 |
| API | FastAPI + Uvicorn, Pydantic | Pydantic 2.x |
| ORM / driver | SQLAlchemy + PyMySQL (+ `cryptography` for MySQL 8 auth) | SQLAlchemy 2.x |
| Database | MySQL | 8.0 |
| Auth | PyJWT (HS256), pwdlib with argon2 | latest |
| Validation extras | `email-validator` (for `EmailStr`) | latest |
| ML | scikit-learn, joblib, numpy, pandas | latest; record sklearn version in model metadata |
| Backend tests | pytest, pytest-cov, httpx | latest |
| Load test | Locust | latest |
| Lint | Ruff (lint + format) | latest |
| Frontend runtime | Node.js | 22 LTS |
| Frontend | React + Vite, plain JavaScript (JSX), no TypeScript | React 18 or 19, Vite current major |
| Routing / charts | React Router, Recharts | current majors |
| Frontend tests | Vitest + `@vitest/coverage-v8`, React Testing Library, `@testing-library/user-event`, `@testing-library/jest-dom`, jsdom, MSW | MSW 2.x (`http` / `HttpResponse` API) |
| E2E | `@playwright/test` (Chromium), optional `@axe-core/playwright` | latest |
| Frontend lint | ESLint + Prettier | latest |
| API collection | Postman collection v2.1, run with Newman | latest |
| Containers | Docker, docker compose v2, nginx (alpine) | nginx stable |
| CI | GitHub Actions | `actions/*` current majors |

## Repository layout

Create exactly this structure; files appear phase by phase.

```text
sentinel/
├── CLAUDE.md
├── README.md
├── docs/SPEC.md
├── .env.example
├── .gitignore                 # .env, __pycache__, node_modules, dist, coverage, reports/*, playwright-report
├── pyproject.toml             # pytest + coverage + ruff config
├── requirements.txt           # runtime deps (pinned)
├── requirements-dev.txt       # test/lint deps (pinned)
├── Dockerfile                 # backend image
├── docker-compose.yml         # db + app + web
├── app/
│   ├── __init__.py
│   ├── main.py                # create_app() factory, lifespan, routers
│   ├── config.py              # Settings from env
│   ├── db.py                  # Base, engine, SessionLocal, get_db
│   ├── models.py              # User, PredictionLog
│   ├── schemas.py             # Pydantic request/response models
│   ├── security.py            # hash/verify password, create/decode token, get_current_user
│   ├── ml.py                  # ModelService: load, verify hash, predict
│   └── routers/
│       ├── health.py          # GET /health
│       ├── users.py           # POST /users, GET /users/me
│       ├── auth.py            # POST /login
│       ├── predict.py         # POST /predict
│       ├── predictions.py     # GET /predictions, GET /predictions/stats
│       └── reports.py         # GET /reports/latest
├── ml/
│   ├── generate_data.py
│   ├── train.py
│   └── data/train.csv
├── models/
│   ├── model_v1.joblib
│   └── model_v1.metadata.json
├── scripts/
│   ├── make_golden.py         # writes tests/regression/golden/predictions_v1.json
│   └── analyze_results.py     # writes reports/summary.json
├── sql/
│   ├── schema.sql             # reference DDL (app uses create_all)
│   └── checks.sql             # analysis queries
├── reports/.gitkeep
├── postman/
│   ├── ai-service.postman_collection.json
│   └── local.postman_environment.json
├── tests/
│   ├── conftest.py
│   ├── unit/
│   ├── integration/
│   ├── api/conftest.py        # overrides client with httpx against BASE_URL
│   ├── regression/golden/predictions_v1.json
│   └── performance/locustfile.py
├── frontend/
│   ├── package.json
│   ├── vite.config.js
│   ├── playwright.config.js
│   ├── eslint.config.js
│   ├── .prettierrc
│   ├── index.html
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── src/
│   │   ├── main.jsx, App.jsx, styles.css
│   │   ├── api/client.js
│   │   ├── auth/token.js, auth/RequireAuth.jsx
│   │   ├── pages/SignIn.jsx, Predict.jsx, History.jsx, Dashboard.jsx
│   │   ├── components/        # NavBar, PredictForm, ResultCard, HistoryTable, LayerChart, StatCard (+ *.test.jsx beside each)
│   │   └── test/setup.js, test/server.js, test/handlers.js
│   └── e2e/
│       ├── fixtures/summary.json
│       └── *.spec.js
└── .github/workflows/ci.yml
```

## Configuration and environment

All backend settings come from environment variables, read once in `app/config.py` (a small class or `pydantic-settings`). Commit `.env.example` with these dev defaults.

| Variable | Default (dev) | Used by |
| --- | --- | --- |
| `DATABASE_URL` | `mysql+pymysql://root:root@127.0.0.1:3306/ai_api` | app |
| `TEST_DATABASE_URL` | `mysql+pymysql://root:root@127.0.0.1:3306/ai_api_test` | pytest integration |
| `JWT_SECRET` | `dev-only-secret-change-me` | app |
| `JWT_EXPIRE_MINUTES` | `30` | app |
| `MODEL_PATH` | `models/model_v1.joblib` | app |
| `MODEL_METADATA_PATH` | `models/model_v1.metadata.json` | app |
| `REPORTS_PATH` | `reports/summary.json` | app (`/reports/latest`) |
| `CORS_ORIGINS` | empty (same-origin via proxy) | app; only set if needed |
| `BASE_URL` | `http://localhost:8000` | pytest `api` layer |
| `E2E_BASE_URL` | `http://localhost:8080` | Playwright |

Local development: run MySQL with `docker compose up -d db`, the API with `uvicorn app.main:create_app --factory --reload`, and the frontend with `npm run dev` (Vite on `:5173`, proxying `/api`).

## Backend: data model

Two SQLAlchemy 2.0 models (typed `Mapped[...]` style). Tables are created in FastAPI's `lifespan` with `Base.metadata.create_all`; no migrations in this project. Keep `sql/schema.sql` as reference DDL matching the models.

```sql
CREATE TABLE users (
  id            INT AUTO_INCREMENT PRIMARY KEY,
  email         VARCHAR(254) NOT NULL,          -- stored trimmed + lower-cased
  full_name     VARCHAR(100) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,          -- argon2 via pwdlib, never plaintext
  created_at    DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  UNIQUE KEY uq_users_email (email)
);

CREATE TABLE prediction_logs (
  id            BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id       INT NOT NULL,
  age           INT NOT NULL,
  income        DECIMAL(12,2) NOT NULL,
  label         VARCHAR(16) NOT NULL,           -- 'approved' | 'rejected'
  probability   DECIMAL(6,4) NOT NULL,
  model_version VARCHAR(16) NOT NULL,
  latency_ms    DECIMAL(8,2) NOT NULL,
  created_at    DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  CONSTRAINT fk_logs_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  INDEX ix_logs_user_time (user_id, created_at),
  INDEX ix_logs_version (model_version)
);
```

- Timestamps are stored in UTC and returned as ISO 8601 strings with a `Z` suffix.
- `DATETIME(6)` gives microsecond precision; history ordering still tie-breaks on `id DESC`, so two rows in the same microsecond can't swap order.
- `sql/checks.sql` holds the analysis queries: label distribution and mean probability per model version, and prediction count per day.

## Backend: API contract

Every endpoint, field, status code and message below is exact; tests, Postman and the frontend are written against it.

### Global rules

- JSON in and out. Request models use `ConfigDict(strict=True, extra="forbid", str_strip_whitespace=True)`. Strict mode makes `"25"` and `true` invalid for an int field (lax mode would coerce `"25"`). Strict floats still accept JSON integers, so `"income": 50000` is valid.
- Validation errors and malformed JSON → FastAPI's default `422` body `{"detail": [ ... ]}`. Do not customise it.
- All other errors → `{"detail": "<message>"}` using the exact messages below.
- Every `401` carries the header `WWW-Authenticate: Bearer`.
- Auth dependency `get_current_user`: `HTTPBearer(auto_error=False)`. A missing header or a non-`Bearer` scheme → `401 "Not authenticated"`. Decode with PyJWT, HS256, requiring `exp` and `sub`. A bad signature, expired token, malformed token or `sub` of a user that doesn't exist → `401 "Invalid or expired token"`.
- Token claims: `sub` = user id as a string, `iat`, `exp` = now + `JWT_EXPIRE_MINUTES`.

### Endpoints

| Method + path | Auth | Success | Errors |
| --- | --- | --- | --- |
| `GET /health` | none | `200` | `503` |
| `POST /users` | none | `201` | `409`, `422` |
| `POST /login` | none | `200` | `401`, `422` |
| `GET /users/me` | bearer | `200` | `401` |
| `POST /predict` | bearer | `200` | `401`, `422`, `503` |
| `GET /predictions` | bearer | `200` | `401`, `422` |
| `GET /predictions/stats` | bearer | `200` | `401` |
| `GET /reports/latest` | bearer | `200` | `401`, `404` |

**`GET /health`**: runs `SELECT 1` and checks the model is loaded.

- `200` `{"status": "ok", "db": "ok", "model": "ok", "model_version": "1.0.0"}`
- `503` `{"status": "degraded", "db": "ok"|"error", "model": "ok"|"error", "model_version": "1.0.0"|null}`

**`POST /users`**

- Body: `email` (`EmailStr`, ≤ 254 chars), `password` (str, 8–128 chars), `full_name` (str, 1–100 chars after trimming).
- Normalise `email` to trimmed lower case before checking uniqueness and storing.
- `201` `{"id": 1, "email": "a@example.com", "full_name": "A User", "created_at": "2026-10-14T16:00:00.000000Z"}`. Never return the password or hash.
- `409` `"Email already registered"`, from the pre-check and also from catching `IntegrityError`, which covers a race.

**`POST /login`**

- Body: `email` (str, 1–254), `password` (str, 1–128). Email is normalised the same way.
- `200` `{"access_token": "<jwt>", "token_type": "bearer", "expires_in": 1800}` (`expires_in` = minutes × 60).
- `401` `"Invalid email or password"`, identical for an unknown email and a wrong password. When the email is unknown, still run a password verify against a fixed dummy hash, so response time doesn't reveal which emails exist.

**`GET /users/me`** → `200` with the same shape as the `POST /users` response.

**`POST /predict`**

- Body: `age` (int, 18–100 inclusive), `income` (float, 0–10,000,000 inclusive, `allow_inf_nan=False`).
- `200` `{"label": "approved"|"rejected", "probability": 0.8312, "model_version": "1.0.0", "latency_ms": 1.73}`.
- `probability` is the model's probability for the returned label (always ≥ 0.5), rounded to 4 decimals. `latency_ms` is `time.perf_counter()` around the model call only, rounded to 2 decimals.
- `503` `"Model unavailable"` if the model failed to load (see the ML model section). Writes exactly one `prediction_logs` row on success; writes nothing on any error.

**`GET /predictions?limit=20&offset=0`**

- `limit`: int 1–100, default 20. `offset`: int ≥ 0, default 0. Out of range → `422`.
- Returns only the caller's rows, ordered `created_at DESC, id DESC`.
- `200` `{"items": [{"id", "age", "income", "label", "probability", "model_version", "latency_ms", "created_at"}], "total": 42, "limit": 20, "offset": 0}`. `income`, `probability` and `latency_ms` are JSON numbers, not strings: convert `Decimal` to `float`.

**`GET /predictions/stats`**: aggregates over all users, no per-user data.

- `200` `{"total": 120, "by_label": {"approved": 64, "rejected": 56}, "by_model_version": [{"model_version": "1.0.0", "count": 120, "avg_probability": 0.7731, "p95_latency_ms": 2.41}]}`
- Both `by_label` keys are always present (0 if none). An empty table gives `total: 0` and `by_model_version: []`.
- p95 uses the nearest-rank method: sort the latencies, take index `ceil(0.95 × n) − 1`. `avg_probability` is rounded to 4 decimals, `p95_latency_ms` to 2.

**`GET /reports/latest`**

- `200` with the parsed contents of the file at `REPORTS_PATH`, returned as-is.
- `404` `"No test report yet"` when the file doesn't exist.

## ML model

A logistic-regression classifier on `age` and `income`, trained once on seeded synthetic data and committed as a versioned file. Accuracy is not a goal; a stable, deterministic contract is.

**`ml/generate_data.py`**: writes `ml/data/train.csv` with columns `age,income,label`.

```python
rng = np.random.default_rng(42)
n = 5000
age = rng.integers(18, 101, n)                      # 18..100 inclusive
income = np.clip(rng.lognormal(np.log(40_000), 0.8, n), 0, 10_000_000).round(2)
score = 0.04 * (age - 59) + 1.2 * (np.log1p(income) - np.log1p(40_000)) + rng.normal(0, 0.5, n)
label = np.where(score > 0, "approved", "rejected")   # roughly balanced classes
```

**`ml/train.py`**

- Pipeline, trained on a DataFrame with columns `age, income`:
  - `ColumnTransformer`: `age` passthrough; `income` through `FunctionTransformer(np.log1p)`
  - `StandardScaler()`
  - `LogisticRegression(random_state=42)`
- Split: `train_test_split(test_size=0.2, random_state=42, stratify=label)`.
- Save `models/model_v1.joblib`, then write `models/model_v1.metadata.json`:

```json
{
  "model_version": "1.0.0",
  "sha256": "<sha256 of model_v1.joblib>",
  "seed": 42,
  "features": ["age", "income"],
  "classes": ["approved", "rejected"],
  "train_rows": 4000,
  "test_accuracy": 0.0,
  "test_roc_auc": 0.0,
  "sklearn_version": "<sklearn.__version__>"
}
```

- Print the accuracy and ROC AUC. Commit the CSV, the model and the metadata. CI never retrains.

**`app/ml.py`: `ModelService`**

- On construction: read the metadata, compute the sha256 of the model file, and raise `ModelIntegrityError` if it doesn't match. Then `joblib.load` the pipeline and expose `version`.
- `predict(age, income) -> (label, probability)`: build a one-row DataFrame, call `predict_proba`, take the argmax class from `classes_`, and return `(label, round(float(p), 4))`.
- Routers get the service through a `get_model` dependency, so tests can override it with a fake that raises or sleeps.

**Loading in the app.** In `lifespan`, construct the `ModelService` and store it on `app.state.model`. If loading fails (missing file or hash mismatch), log the error and set it to `None`, so the app still starts. Then `/health` returns `503` with `"model": "error"`, and `/predict` returns `503 {"detail": "Model unavailable"}`.

**`scripts/make_golden.py`**: writes `tests/regression/golden/predictions_v1.json` as `{"model_version": "1.0.0", "cases": [{"age", "income", "label", "probability"}]}`.

- 30 grid cases: ages `[18, 25, 40, 59, 75, 100]` × incomes `[0, 15000, 40000, 120000, 10000000]`.
- 20 random cases from `np.random.default_rng(7)` within the valid ranges.
- Regenerate it only when the model version changes, in its own commit.

## Frontend

A React + Vite single-page app in plain JavaScript, styled with one plain CSS file. It talks only to the API through `/api`. All visible text below is exact, because the component and E2E tests locate elements by it.

### Shell and routes

- Browser tab title and header name: Sentinel, with the subtitle **Loan Approval AI**. A `<nav>` holds the links **Predict**, **History**, **Dashboard** and a **Sign out** button, all shown only when signed in.
- Routes: `/signin` (public); `/predict`, `/history`, `/dashboard` (wrapped in `RequireAuth`); `/` redirects to `/predict`; unknown paths redirect to `/predict`.
- `RequireAuth`: no token → `<Navigate to="/signin" replace />`.

### Pages

| Page | Elements (exact labels and text) | Behaviour |
| --- | --- | --- |
| `/signin` | Inputs **Email**, **Password**; submit **Sign in**. Toggle link-button **Need an account? Create one** switches to create mode: inputs **Full name**, **Email**, **Password**; submit **Create account**; toggle **Have an account? Sign in** | Sign in → `POST /login`, store token, go to `/predict`. Create → `POST /users`, then log in automatically. Errors in a `role="alert"` element: 401 → **Invalid email or password**; 409 → **Email already registered**. `?expired=1` shows **Your session has expired. Please sign in again.** |
| `/predict` | Number inputs **Age**, **Income**; submit **Predict** (shows **Predicting…** and is disabled while loading). Result card in an `aria-live="polite"` region: label (`data-testid="result-label"`, text exactly `approved`/`rejected`, capitalised only via CSS), probability (`data-testid="result-probability"`, e.g. **83.1%**), **Model 1.0.0**, **1.73 ms** | Client checks before sending: **Age must be a whole number between 18 and 100**; **Income must be between 0 and 10,000,000**. Invalid → no request. Server 422 → message shown under the field named by the last element of `detail[].loc`. Other failures → alert **Something went wrong. Please try again.**, inputs keep their values |
| `/history` | Table with caption **Your predictions**; columns **Time, Age, Income, Result, Probability, Model**; buttons **Previous**, **Next**; text **Showing 1–20 of 42** | 20 per page via `GET /predictions`. Time in the browser's locale; income with `Intl.NumberFormat("en-US")`; probability as a percentage with 1 decimal. Empty → **No predictions yet.** plus a link **Make your first prediction** |
| `/dashboard` | Section **Latest test run**: stat cards **Pass rate**, **Backend coverage**, **Frontend coverage**, **Last run** (time + short commit); stacked bar chart per layer; table **Results by layer** (Layer, Passed, Failed, Skipped, Duration); table **Slowest tests**; table **Endpoint latency** (Endpoint, p50 ms, p95 ms, Req/s). Section **Service stats**: cards **Total predictions**, **Approved**, **Rejected**; table **By model version** | Report `404` → **No test run yet.** in place of the section. Pass rate = passed ÷ (passed + failed) over all layers. A layer with failures shows a **Failing** badge in its table row. A missing value (e.g. `null` coverage) shows **—** |

The **Results by layer** table always renders next to the chart. It gives screen-reader users the data, and gives tests something to assert on, because Recharts' `ResponsiveContainer` has zero size in jsdom.

### API client and auth

- `src/api/client.js` is the only module that calls `fetch`. It exports `api.createUser`, `api.login`, `api.me`, `api.predict`, `api.listPredictions(limit, offset)`, `api.stats`, `api.latestReport`.
- URLs are built as `new URL("/api" + path, window.location.origin)`. Absolute URLs work in the browser and let MSW intercept requests in jsdom.
- Non-2xx → throw `ApiError {status, detail}`; network failure → `ApiError {status: 0, detail: "Network error"}`.
- A `401` on an authenticated call clears the token and calls the handler registered with `setUnauthorizedHandler`. The app registers one that navigates to `/signin?expired=1`.
- `src/auth/token.js`: `getToken`, `setToken`, `clearToken`, using `sessionStorage` key `ai_api_token`. Wrap every storage access in try/catch with an in-memory fallback.

### Look and accessibility

- One `styles.css` with CSS variables, system font stack, content max-width 960px, readable down to 360px wide. Approved shown in green, rejected in red, always with the text as well, never colour alone.
- Every input has a `<label>`; field errors are linked with `aria-describedby` and set `aria-invalid`; tables have captions; focus is visible.
- `data-testid` is used only for `result-label` and `result-probability`. Everything else is found by role, label or text.

### Config files

- `package.json` scripts: `dev`, `build`, `preview`, `test` (`vitest run`), `coverage` (`vitest run --coverage`), `lint` (`eslint .`), `format` (`prettier --write .`), `e2e` (`playwright test`).
- `vite.config.js`: React plugin; dev proxy `/api` → `http://localhost:8000` with the `/api` prefix stripped. Vitest settings: `environment: "jsdom"`, `setupFiles: "./src/test/setup.js"`, `include: ["src/**/*.test.{js,jsx}"]` (keeps the Playwright specs out, since Vitest's default pattern would also match `*.spec.js`), coverage `provider: "v8"`, `reporter: ["text", "json-summary", "html"]`, `thresholds: { lines: 80 }`.
- `src/test/setup.js`: import `@testing-library/jest-dom/vitest`; start the MSW server with `onUnhandledRequest: "error"`; reset handlers and clear `sessionStorage` after each test; close the server after all.
- `playwright.config.js`: `testDir: "./e2e"`, `baseURL: process.env.E2E_BASE_URL || "http://localhost:8080"`, Chromium only, `retries: process.env.CI ? 1 : 0`, `trace: "on-first-retry"`, reporters `html` and `junit` with output `../reports/playwright.xml`.

## Test suite requirements

Every bullet below is at least one test; most are parametrized cases. Write them in the phase that builds the code they cover.

### Backend setup

- `pyproject.toml`: `testpaths = ["tests"]`, `addopts = "-ra --strict-markers"`, markers `unit, integration, api, regression, performance`. Coverage: `source = ["app"]`, `branch = true`, `fail_under = 85`, `show_missing = true`.
- `tests/conftest.py` auto-marks each test by its folder name in `pytest_collection_modifyitems`, so tests carry no marker decorators.
- `create_app(settings=None)` accepts a `Settings` object, so tests can point it at `TEST_DATABASE_URL`, a temporary `REPORTS_PATH` and a tampered model path.
- Fixtures:
  - `engine` (session scope): `drop_all` then `create_all` on the test DB.
  - `db_session`: connection + outer transaction + `sessionmaker(join_transaction_mode="create_savepoint")`, rolled back after each test.
  - `client`: `TestClient(create_app(test_settings))` with `get_db` overridden to yield `db_session`.
  - `model`: a session-scoped `ModelService`.
  - `user_payload`: unique email per test (`uuid4().hex[:8]`), password `S3cure-pass`.
  - `auth_headers`: creates the user, logs in, returns the Bearer header.
  - `make_token(sub, minutes)`: forges tokens, including expired ones and ones signed with the wrong secret.
- `tests/api/conftest.py` overrides `client` with `httpx.Client(base_url=os.environ["BASE_URL"])`.
- Send malformed JSON and `NaN`/`Infinity` as raw bytes (`content=b'...'` with a JSON content type), so the client can't fix them.

### `tests/unit/`

- **`test_security.py`**: hash ≠ password and starts with `$argon2`; verify true/false; token round-trip returns `sub`; expired, wrong-secret and tampered tokens all raise.
- **`test_schemas.py`**: for `UserCreate`, `LoginRequest` and `PredictRequest`, valid cases pass. These are rejected: `age` as `"25"`, `true`, `25.5`; `income` as `"50k"`, NaN, Infinity; age 17/101; income -1/10,000,001; password of 7 and 129 chars; extra fields. Email is trimmed and lower-cased.
- **`test_ml.py`**: valid label; probability in \[0.5, 1\]; 50 identical calls give identical output; a tampered model copy in `tmp_path` raises `ModelIntegrityError`; approval probability never decreases as income rises (ages 18, 59 and 100 across 10 incomes); the log-income coefficient is positive.
- **`test_training_data.py`** (pandas): 5,000 rows; no nulls; ages are integers in 18–100; income in 0–10M; labels in {approved, rejected}; each class between 30% and 70%.
- **`test_percentile.py`**: nearest-rank p95 helper: `[1..20]` → 19, `[5]` → 5, `[]` → `None`.

### `tests/integration/` (TestClient + real MySQL)

- **Users:** `201` shape with no password field; the DB row's hash starts with `$argon2`; duplicate → `409`; the same email in a different case or with surrounding spaces → `409`; a raw SQL insert of a duplicate raises `IntegrityError`; passwords of 8 and 128 chars accepted, 7 and 129 → `422`; `is_admin: true` → `422`; malformed JSON → `422`.
- **Auth:** login `200`, token decodes, `expires_in == 1800`; a wrong password and an unknown email both give `401` with identical bodies; missing fields → `422`. `/users/me` works with a valid token. These all give `401` plus `WWW-Authenticate: Bearer`: no header, a `Basic` scheme, a bad signature, an expired token, a token for a user that no longer exists.
- **Predict:** valid `200` with an exact key set; boundaries age 17/18/100/101 and income -1/0/10M/10M+1 give the expected `422`/`200`; wrong types; each field missing; `{}`; `NaN` and `Infinity` as raw bytes; malformed JSON; extra field; no token → `401`. Each `200` writes exactly one log row matching the response; `401`/`422` write none. A model that fails to load → `503`.
- **History and stats:** newest first; `total` matches `COUNT(*)`; `limit` 0/101 and `offset` -1 → `422`; pagination returns disjoint pages; user B never sees user A's rows. Stats equal values computed in the test from seeded rows with known latencies; an empty table returns the empty shape.
- **Reports:** missing file → `404 "No test report yet"`; a present file is returned exactly.
- **Health:** `200` shape; `503` with `"model": "error"` when the model is missing; `503` with `"db": "error"` when `get_db` is overridden with a session that raises `OperationalError`.

### `tests/api/` (black-box against the running stack)

Unique emails per run. Health `200`; register → login → predict → the prediction appears first in `/predictions`; duplicate `409`; wrong password `401`; predict without a token `401`; invalid age `422`; `/reports/latest` is `200` or `404`, and if `200` the body has `layers`.

### `tests/regression/`

- **Golden set:** every case in `predictions_v1.json` gives the same label and a probability within `1e-6`.
- **Schema snapshot:** the exact key sets of every endpoint's success response. Adding or removing a field must fail a test.
- **Version agreement:** metadata `model_version` = `/health` = `/predict` response.
- **Reproducibility:** retraining in memory from `train.csv` with seed 42 gives coefficients equal to the loaded model's within `1e-9`.
- **Bug repros:** `test_bug_<n>_<slug>.py`, one per real bug found, each with a docstring describing the bug.

### `tests/performance/`

- Model p95 latency < 20 ms over 200 calls after 10 warm-up calls.
- API `POST /predict` p95 < 200 ms over 100 calls via httpx against `BASE_URL`.
- Measure on the CI runner first, then set each budget to about 2× the observed p95. Never raise a budget without a commit explaining why.
- `locustfile.py`: on start, register and log in; tasks `predict` (weight 5) and `history` (weight 1); `wait_time = between(0.5, 2)`.

### Frontend component tests (Vitest + React Testing Library + MSW)

- **SignIn:** successful sign-in stores the token and navigates; 401 shows **Invalid email or password**; create mode posts to `/users` then logs in; 409 shows **Email already registered**; `?expired=1` message.
- **PredictForm:** out-of-range age or income shows the exact message and sends zero requests; valid submit shows **Predicting…** then the result with **83.1%**; server 422 shows the message under the right field; 500 and network failure show the alert and keep the inputs.
- **ResultCard:** probabilities 0.5 and 1 render; the label text stays lowercase in the DOM.
- **HistoryTable / History:** empty state with link; 20 rows plus **Showing 1–20 of 42**; **Next** disabled on the last page; **Previous** disabled on the first.
- **Dashboard:** report `404` → **No test run yet.**; a layer with failures shows **Failing**; pass rate computed correctly; `null` coverage shows **—**; service stats render.
- **api/client and auth:** the Bearer header is attached; a 401 clears the token and calls the unauthorized handler; network failure gives status 0; `RequireAuth` redirects without a token; **Sign out** clears the token and goes to `/signin`.
- Coverage ≥ 80% lines on `src/`.

### End-to-end (`frontend/e2e/`, Playwright, Chromium)

1. **auth.spec.js:** create an account → lands on `/predict`; sign out → `/signin`; a wrong password shows the error.
2. **predict-history.spec.js:** a valid prediction shows the result; History's first row contains the same age and label.
3. **validation.spec.js:** age 17 shows the inline error, and the History count is unchanged.
4. **session.spec.js:** clearing `sessionStorage` then navigating to `/history` redirects to `/signin`.
5. **dashboard.spec.js:** with the seeded fixture `summary.json`, the dashboard shows its exact pass rate, coverage and a **Failing** badge for the fixture's failing layer.
6. Optional **a11y.spec.js:** `@axe-core/playwright` on each page with zero serious or critical violations.

Use role and label locators plus the two test ids. No `waitForTimeout`. Each test signs up its own unique user.

### Postman collection

- Folders: Health, Users, Auth, Predict, History. `{{baseUrl}}` comes from `local.postman_environment.json` (`http://localhost:8000`).
- **Create user**'s pre-request script sets a unique `email` collection variable. **Login**'s test script stores `token` as a collection variable for later requests.
- Each request has `pm.test` assertions on status and key fields. Include the negative requests: duplicate user `409`, wrong password `401`, predict without a token `401`, age 17 `422`.

## Results analysis and dashboard data

`scripts/analyze_results.py` turns raw test output into the one `reports/summary.json` the dashboard reads. It uses pandas, never crashes on missing inputs, and is itself unit-tested.

**Usage:** `python scripts/analyze_results.py --reports-dir reports --out reports/summary.json` writes the JSON and prints a Markdown summary table to stdout (CI appends it to the job summary).

| Input (all optional) | Read as | Feeds |
| --- | --- | --- |
| `reports/*.xml` from pytest (`junit.xml`, `api.xml`, `performance.xml`) | JUnit; layer = second part of the testcase `classname` (`tests.unit.test_x` → `unit`) | `layers`, `slowest_tests` |
| `reports/vitest.xml` | JUnit; layer `frontend` | `layers`, `slowest_tests` |
| `reports/playwright.xml` | JUnit; layer `e2e` | `layers`, `slowest_tests` |
| `reports/newman.xml` | JUnit; layer `postman` | `layers` |
| `coverage.xml` | root `line-rate` × 100 | `coverage.backend_pct` |
| `frontend/coverage/coverage-summary.json` | `total.lines.pct` | `coverage.frontend_pct` |
| `reports/locust_stats.csv` | rows except `Aggregated`; endpoint = `Type + " " + Name`; columns `50%`, `95%`, `Requests/s` | `latency` |

A testcase with a `<failure>` or `<error>` child counts as failed, `<skipped>` as skipped, anything else as passed. Missing inputs give `null` fields or empty lists.

**Output schema**

```json
{
  "generated_at": "2026-11-02T20:31:05Z",
  "commit": "a1b2c3d",
  "layers": [
    { "name": "unit", "passed": 42, "failed": 0, "skipped": 1, "duration_s": 3.1 }
  ],
  "coverage": { "backend_pct": 88.4, "frontend_pct": 83.0 },
  "slowest_tests": [{ "name": "tests.performance.test_latency::test_model_p95", "layer": "performance", "duration_s": 0.84 }],
  "latency": [{ "endpoint": "POST /predict", "p50_ms": 12, "p95_ms": 41, "rps": 180.2 }]
}
```

- `generated_at` is UTC. `commit` is the first 7 characters of `GITHUB_SHA`, else `git rev-parse --short HEAD`, else `null`.
- `slowest_tests` holds the top 10 by duration. Layers are ordered unit, integration, api, regression, performance, frontend, e2e, postman, keeping only the ones present.
- **Tests:** `tests/unit/test_analyze_results.py` runs the script on small fixture XML, JSON and CSV files in `tests/unit/fixtures/` and asserts the exact output, including the all-inputs-missing case.

**E2E dashboard fixture** (`frontend/e2e/fixtures/summary.json`). Use exactly these layer counts, so the dashboard test can assert fixed values:

| Layer | Passed | Failed | Skipped |
| --- | --- | --- | --- |
| unit | 40 | 0 | 1 |
| integration | 55 | 0 | 0 |
| api | 12 | 0 | 0 |
| regression | 9 | 1 | 0 |
| frontend | 38 | 0 | 0 |
| e2e | 5 | 0 | 0 |

Coverage backend 88.4, frontend 83.0, commit `abc1234`. The dashboard must then show pass rate **99.4%** (159 ÷ 160), **88.4%** and **83.0%**, and a **Failing** badge on regression. Percentages are formatted with one decimal.

## Docker, compose and GitHub Actions

Three containers (db, app, web) run the same way locally and in CI. GitHub Actions runs lint, then backend and frontend in parallel, then e2e; performance runs nightly and on manual dispatch.

**`Dockerfile`** (backend)

```dockerfile
FROM python:3.12-slim
WORKDIR /srv
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ app/
COPY models/ models/
EXPOSE 8000
CMD ["uvicorn", "app.main:create_app", "--factory", "--host", "0.0.0.0", "--port", "8000"]
```

**`frontend/Dockerfile`**

```dockerfile
FROM node:22-alpine AS build
WORKDIR /web
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:stable-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /web/dist /usr/share/nginx/html
```

**`frontend/nginx.conf`**: the trailing slash on `proxy_pass` strips `/api`; `try_files` makes deep links such as `/history` survive a refresh.

```nginx
server {
  listen 80;
  root /usr/share/nginx/html;
  location /api/ { proxy_pass http://app:8000/; }
  location / { try_files $uri /index.html; }
}
```

**`docker-compose.yml`**

```yaml
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: ai_api
    ports: ["3306:3306"]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-proot"]
      interval: 5s
      retries: 20
  app:
    build: .
    environment:
      DATABASE_URL: mysql+pymysql://root:root@db:3306/ai_api
      JWT_SECRET: dev-only-secret-change-me
      REPORTS_PATH: /srv/reports/summary.json
    volumes:
      - ./reports:/srv/reports:ro
    ports: ["8000:8000"]
    depends_on:
      db: { condition: service_healthy }
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 5s
      retries: 10
  web:
    build: ./frontend
    ports: ["8080:80"]
    depends_on:
      app: { condition: service_healthy }
```

For local integration tests, create the test database once: `docker compose exec db mysql -uroot -proot -e "CREATE DATABASE IF NOT EXISTS ai_api_test"`.

**`.github/workflows/ci.yml`**

```yaml
name: CI
on:
  push:
  pull_request:
  schedule:
    - cron: "30 20 * * *"      # 02:00 IST nightly
  workflow_dispatch:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install ruff
      - run: ruff check . && ruff format --check .
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm, cache-dependency-path: frontend/package-lock.json }
      - run: npm ci && npm run lint && npx prettier --check .
        working-directory: frontend

  backend:
    needs: lint
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: ai_api_test
        ports: ["3306:3306"]
        options: >-
          --health-cmd="mysqladmin ping -h 127.0.0.1 -proot"
          --health-interval=5s --health-timeout=5s --health-retries=20
    env:
      TEST_DATABASE_URL: mysql+pymysql://root:root@127.0.0.1:3306/ai_api_test
      JWT_SECRET: ci-only-secret
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - name: Unit (fail fast)
        run: pytest -m unit -x -q
      - name: Unit + integration + regression with coverage
        run: >
          pytest -m "unit or integration or regression"
          --cov=app --cov-report=xml --cov-report=term-missing --cov-fail-under=85
          --junitxml=reports/junit.xml
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: backend-reports
          path: |
            reports/
            coverage.xml

  frontend:
    needs: lint
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm, cache-dependency-path: frontend/package-lock.json }
      - run: npm ci
      - name: Component tests with coverage
        run: >
          npx vitest run --coverage
          --reporter=default --reporter=junit --outputFile.junit=../reports/vitest.xml
      - run: npm run build
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: frontend-reports
          path: |
            reports/
            frontend/coverage/

  e2e:
    needs: [backend, frontend]
    runs-on: ubuntu-latest
    env:
      BASE_URL: http://localhost:8000
      E2E_BASE_URL: http://localhost:8080
    steps:
      - uses: actions/checkout@v4
      - name: Seed the dashboard with the fixture report
        run: mkdir -p reports && cp frontend/e2e/fixtures/summary.json reports/summary.json
      - run: docker compose up -d --build --wait
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - name: Black-box API tests
        run: pytest -m api --junitxml=reports/api.xml
      - name: Postman collection via Newman
        run: >
          npx --yes newman run postman/ai-service.postman_collection.json
          -e postman/local.postman_environment.json
          --env-var baseUrl=$BASE_URL
          --reporters cli,junit --reporter-junit-export reports/newman.xml
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm, cache-dependency-path: frontend/package-lock.json }
      - name: Playwright end-to-end
        working-directory: frontend
        run: |
          npm ci
          npx playwright install --with-deps chromium
          npx playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with: { name: playwright-report, path: frontend/playwright-report/ }
      - if: failure()
        run: docker compose logs app web
      - if: always()
        run: docker compose down -v

  performance:
    if: github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    env:
      BASE_URL: http://localhost:8000
    steps:
      - uses: actions/checkout@v4
      - run: docker compose up -d --build --wait
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest -m performance --junitxml=reports/performance.xml
      - run: >
          locust -f tests/performance/locustfile.py --headless
          -u 50 -r 10 -t 1m --host $BASE_URL --csv reports/locust
      - run: python scripts/analyze_results.py --reports-dir reports --out reports/summary.json >> $GITHUB_STEP_SUMMARY
      - uses: actions/upload-artifact@v4
        with: { name: perf-reports, path: reports/ }
```

After the first green run, protect `main` in the GitHub repo settings: require `lint`, `backend`, `frontend` and `e2e` to pass before merging.

## Build phases

Eight phases, each sized for one Claude Code session. Every phase ends with all of its acceptance checks passing, plus everything from earlier phases still passing, then a commit. `ci.yml` grows with the phases: add a job only once the code it tests exists.

1. **Scaffold, database, model, health**
   - Build: repo layout, `pyproject.toml`, `requirements*.txt`, `.env.example`, `.gitignore`, `CLAUDE.md`, README stub; `config.py`, `db.py`, `models.py`, `create_app` with `lifespan`; `ml/generate_data.py`, `ml/train.py`, committed model + metadata; `ModelService`; `GET /health`; compose `db` service; `ci.yml` with `lint` (Python part) and `backend`.
   - Tests: `test_training_data.py`, `test_ml.py`, `test_percentile.py`, health integration tests (`200`, model `503`, DB `503`).
   - Acceptance: `python ml/train.py` prints accuracy and AUC · `pytest -m "unit or integration"` green · coverage ≥ 85% · `ruff check . && ruff format --check .` clean · `curl localhost:8000/health` returns the exact `200` body.
2. **Users and auth**
   - Build: `security.py`, `schemas.py` (user and login models), `POST /users`, `POST /login`, `GET /users/me`, `get_current_user`.
   - Tests: `test_security.py`, the user/login parts of `test_schemas.py`, Users and Auth integration tests.
   - Acceptance: all backend tests green · coverage ≥ 85% · a manual check in `/docs` that register → login → `/users/me` works.
3. **Predict, history, stats, reports**
   - Build: `POST /predict` with logging, `GET /predictions`, `GET /predictions/stats`, `GET /reports/latest`, `sql/schema.sql`, `sql/checks.sql`.
   - Tests: the predict part of `test_schemas.py`; Predict, History and stats, and Reports integration tests, including the user A / user B isolation test.
   - Acceptance: all backend tests green · coverage ≥ 85% · every row of the endpoint table returns its documented codes.
4. **Regression suite, Docker, black-box API, Postman**
   - Build: `scripts/make_golden.py` + committed golden file; backend `Dockerfile`; compose `app` service; `tests/api/`; Postman collection + environment; the API and Newman steps (backend only for now, in an `e2e` job without Playwright, with needs: \[backend\] until the frontend job exists).
   - Tests: all of `tests/regression/` and `tests/api/`.
   - Acceptance: `docker compose up -d --build --wait` healthy · `pytest -m api` green against it · `npx newman run postman/...` green · `pytest -m regression` green · CI green.
5. **Performance and results analysis**
   - Build: performance tests, `locustfile.py`, `scripts/analyze_results.py` with its fixtures and unit test, the `performance` job.
   - Acceptance: `pytest -m performance` green against the stack · a 1-minute headless Locust run writes CSVs · `analyze_results.py` writes a valid `summary.json` from real reports and prints the Markdown table · its unit test is green.
6. **Frontend: shell, sign-in, predict**
   - Build: Vite + React (JS) app, ESLint + Prettier, `styles.css`, `api/client.js`, `auth/token.js`, `RequireAuth`, NavBar, SignIn, Predict page with `PredictForm` + `ResultCard`, MSW test setup; the `frontend` CI job and the JS part of `lint`.
   - Tests: the SignIn, PredictForm, ResultCard, client and auth component tests.
   - Acceptance: `npm run lint` and `npm test` green · `npm run coverage` ≥ 80% lines · `npm run build` succeeds · manual: with the API running, `npm run dev` lets you sign up and get a prediction.
7. **Frontend: history and dashboard**
   - Build: History page with pagination and empty state; Dashboard with stat cards, Recharts layer chart, the **Results by layer** table, the slowest-tests and latency tables, and service stats; `frontend/e2e/fixtures/summary.json` exactly as specified.
   - Tests: the History and Dashboard component tests.
   - Acceptance: all frontend tests green · coverage ≥ 80% · manual: copying the fixture to `reports/summary.json` shows 99.4% pass rate and a **Failing** badge on regression.
8. **End-to-end and polish**
   - Build: `frontend/Dockerfile`, `nginx.conf`, compose `web` service, `playwright.config.js`, the five E2E specs (plus the optional a11y spec), Playwright added to the `e2e` job; finish the README.
   - Acceptance: `docker compose up -d --build --wait` brings up all three services · `npx playwright test` green locally · the full CI workflow is green on a push · the README run instructions work from a fresh clone.

## Definition of done

The project is done when a stranger can clone it, run a handful of commands, and watch every layer pass, and when a deliberately broken change fails CI.

- [ ] All eight phases committed; the CI workflow is green on `main`
- [ ] Every bullet in the test suite requirements has at least one test
- [ ] Backend coverage ≥ 85% and frontend coverage ≥ 80% lines, both enforced in CI
- [ ] `docker compose up -d --build --wait`, then `pytest`, `npx newman run ...` and `npx playwright test` pass from a fresh clone
- [ ] The nightly `performance` job ran green at least once and wrote a job summary
- [ ] The dashboard shows a real `summary.json` produced by `analyze_results.py` from a local run
- [ ] Branch protection on `main` requires `lint`, `backend`, `frontend`, `e2e`; one deliberately failing PR shows the gate blocking the merge
- [ ] README contains: one-paragraph overview, architecture diagram, setup in ≤ 5 commands, how to run each test layer, the test matrices, CI badge, a dashboard screenshot, and a "bugs the suite caught" section linking to the `test_bug_*` files
- [ ] No test is skipped, `xfail`ed or loosened compared with this spec
