# CLAUDE.md

## Project overview

**Bookly** is a multi-tenant booking platform for small businesses such as salons, clinics and gyms. Each business has a dashboard (owner and staff roles) and a public booking page at `/b/{slug}`, where customers book appointments as guests.

The full specification is in [docs/SPEC.md](docs/SPEC.md), and it is the source of truth for scope, rules, the data model, the API and the phases. Read the relevant sections before starting work on a phase.

### Stack

- **Backend:** Python, FastAPI, SQLAlchemy 2.0 (sync) with psycopg 3, Alembic, **PostgreSQL 18**, pytest. Dependencies are managed with `uv`; ruff handles linting and mypy handles type checking.
- **Password hashing:** `pwdlib` (bcrypt) or the `bcrypt` package directly. **Never use `passlib`**: it is unmaintained and breaks with recent `bcrypt`.
- **Frontend:** **React 19**, TypeScript (strict), Vite, React Router, TanStack Query, React Hook Form + Zod, and Vitest + React Testing Library + MSW for tests.
- **Infra:** Docker Compose runs Postgres from phase 1; Docker Desktop must be installed first. The full stack is containerized in phase 6, along with GitHub Actions CI. Phase 7 adds an AI natural-language booking assistant.
- **UI language:** English.

### Layout

- `backend/`: FastAPI app (`app/`), migrations (`alembic/`) and tests (`tests/unit`, `tests/integration`)
- `frontend/`: Vite React app
- `docs/`: specification

## About the developer

The developer is experienced with **Flutter/Dart** and is learning Python and React through this project. It is a CV portfolio piece, so it should be built the way a professional team would: clean structure, typed code, tests, small focused commits and pull requests.

## Rules

1. **Explain new concepts briefly.** The first time you introduce a Python or React concept (for example decorators, dependency injection with `Depends`, context managers, Pydantic models, hooks, `useEffect`, context or TanStack Query), give a short 1–3 sentence explanation. When it helps, compare it to the Flutter/Dart equivalent, for example "`useState` ≈ `setState` in a StatefulWidget". Don't explain the same concept again.
2. **Run the tests after code changes.** Run the relevant suite after every change (`pytest` in `backend/`, `npm test` in `frontend/`), plus the linters and type checkers once they are set up. Report the results honestly, including any failures.
3. **Never commit or push without asking first.** Propose a commit message and wait for explicit approval each time.
4. **Never commit secrets.** `.env` files, keys and tokens must never be committed or printed in full. Only `.env.example`, with placeholder values, is committed. Check `git status` and the staged diff before any commit.

## Engineering conventions

- **Tenant isolation is non-negotiable.** Dashboard endpoints take `business_id` from the verified JWT only. All database access goes through repositories that require `business_id`. Requests for another tenant's resources return 404. Every new dashboard route must be added to the tenant-isolation test matrix (see SPEC §4.4).
- **Time:** store UTC (`timestamptz`), and convert to the business time zone with `zoneinfo`. Never use naive datetimes.
- **Validation:** Pydantic models use `extra="forbid"`. Public endpoints are rate-limited (SPEC §4.5).
- **Tokens in the browser:** the access token is kept in memory only, and the refresh token is in an HttpOnly cookie. Never use `localStorage` for tokens (SPEC §4.2).
- **Tests:** integration tests run against real PostgreSQL 18, not SQLite. New behavior comes with tests.
- **Git:** use feature branches and pull requests into `main`, with Conventional Commits (`feat:`, `fix:`, `test:`, `docs:`, `chore:`).

## Commands

_To be filled in during phase 1 (setup)._
