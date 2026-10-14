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