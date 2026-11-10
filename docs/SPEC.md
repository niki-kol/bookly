# Bookly — Product & Technical Specification

> Status: Draft v1 · Last updated: 2026-11-10

Bookly is a multi-tenant booking platform for small businesses such as hair salons, clinics and gyms. Each business gets:

- a **dashboard**, where owners and staff manage services, schedules and bookings
- a **public booking page**, where customers book appointments without creating an account

The UI language is **English**.

---

## 1. MVP scope

The MVP is the smallest version that is useful end to end. It includes:

**Businesses**
- An owner signs up, which creates both the business and the owner's account.
- Each business has a name, a unique URL slug and an IANA time zone (for example `Europe/Athens`).

**Services**
- Name, duration in minutes, price and an active/inactive flag.

**Staff**
- Staff members are the people who perform services.
- Each staff member has:
  - the services they offer
  - weekly working hours, with multiple intervals per day so breaks are possible
  - time-off blocks
- A staff member can optionally have a login.

**Bookings**
- A booking is one customer, one service and one staff member at a specific time.
- Owners can view, create and cancel bookings from the dashboard.
- Customers can book and cancel from the public page.

**Availability**
- The system calculates free slots per service, staff member and date.

**Roles**
- **Owner:** full access to their own business.
- **Staff:** read access to their own schedule and bookings.

**Guest booking**
- Customers enter their name, email and phone. They do not need an account.

**Cancel link**
- Every booking gets a private, unguessable cancel link.

**Notifications**
- Confirmation and cancellation "emails" are written to the server log through an `EmailSender` interface. A real email provider can be added later.

**Tenant isolation**
- Business A can never read or change Business B's data. Automated tests enforce this (see §4.4).

**Public endpoint protection**
- The public endpoints use rate limiting and strict input validation (see §4.5).

---

## 2. Main user flows

### 2.1 Owner onboarding
1. The owner opens `/signup` and enters their business name, slug, time zone, email and password.
2. The backend creates the `business`, the owner `user` and a `staff` record for the owner, so a solo business works right away.
3. The owner is logged in and redirected to the dashboard.

### 2.2 Owner sets up the business
1. The owner creates services, for example "Haircut, 30 min, €20".
2. The owner adds staff members and assigns services to them.
3. The owner sets weekly working hours for each staff member, plus any time off.
4. Optionally, the owner creates a login for a staff member (email plus an initial password).
5. The owner copies the public link `https://<host>/b/<slug>` to share with customers.

### 2.3 Customer books an appointment
1. The customer opens `/b/<slug>` and sees the business name and its active services.
2. The customer picks a service, then a staff member or "Any available".
3. The customer picks a date and sees free time slots in the business's time zone.
4. The customer picks a slot and enters their name, email and phone.
5. The backend re-checks availability and creates the booking in a single transaction.
6. The customer sees a confirmation page with the cancel link. The confirmation "email" is logged.

### 2.4 Customer cancels
1. The customer opens `/b/<slug>/booking/<token>` and sees the booking summary.
2. The customer clicks "Cancel booking". The booking status becomes `cancelled` and the slot becomes free again.

### 2.5 Staff checks their schedule
1. A staff member logs in and sees their own upcoming bookings in a day or week view.
2. Staff cannot edit services, other staff members or business settings.

### 2.6 Owner manages bookings
1. The owner sees all bookings, filtered by date range and staff member.
2. The owner can create a booking manually, for example for a phone booking. The same conflict rules apply.
3. The owner can cancel any booking.

---

## 3. Booking and conflict rules

| # | Rule |
|---|------|
| R1 | All timestamps are stored in **UTC** (`timestamptz`). Working hours are stored as local wall-clock times in the **business time zone** and converted with `zoneinfo`, which handles daylight saving time (DST) correctly. |
| R2 | Bookings use half-open intervals `[starts_at, ends_at)`. A booking ending at 10:00 does **not** conflict with one starting at 10:00. |
| R3 | `ends_at = starts_at + service.duration_minutes`. The client never sends `ends_at`. |
| R4 | Slot start times are aligned to a **15-minute grid** in the business time zone. |
| R5 | A booking must fit entirely inside one of the staff member's working-hour intervals for that local date. |
| R6 | A booking must not overlap any of the staff member's time-off blocks. |
| R7 | A booking must not overlap another **confirmed** booking for the same staff member. Cancelled bookings never block a slot. |
| R8 | The staff member must offer the chosen service, and both must be active. |
| R9 | Lead time: customers cannot book less than **1 hour** ahead. Horizon: customers cannot book more than **60 days** ahead. Owners creating bookings manually are exempt from the lead-time rule. |
| R10 | "Any available" assigns the staff member with the fewest bookings that day. Ties are broken by staff `id`. |
| R11 | **Race condition safety:** R7 is enforced in the database with a PostgreSQL `EXCLUDE USING gist` constraint on `(staff_id WITH =, tstzrange(starts_at, ends_at) WITH &&) WHERE status = 'confirmed'`. If two customers book the same slot at the same moment, one succeeds and the other gets `409 Conflict`. The application checks are there for clear error messages; the constraint is the real guarantee. |
| R12 | Customers can cancel up until the booking's start time. A booking that has already started cannot be cancelled through the public link. |
| R13 | Each booking stores a copy of the service price (`price_cents`) at the time of booking, so editing the service later does not change past bookings. |

---

## 4. Security design

### 4.1 Authentication
- Passwords are hashed with **bcrypt**, using **`pwdlib`** (with its bcrypt hasher) or the `bcrypt` package directly. **`passlib` is not used**: it is unmaintained and breaks with recent `bcrypt` versions. Plain-text passwords are never logged or stored.
- **Access token:** a JWT signed with HS256 that expires after **15 minutes**. Its claims are `sub` (user id), `business_id`, `role` and `exp`.
- **Refresh token:** a random opaque string that expires after **7 days**. Only its SHA-256 hash is stored in the `refresh_tokens` table. The token is rotated on every use, and reuse of an old token revokes the whole token family.

### 4.2 How tokens are stored in the browser
- The **access token is kept only in memory** (a React context/module variable). It is never put in `localStorage` or `sessionStorage`, because any injected script (XSS) can read those.
- The **refresh token is stored in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie** scoped to `Path=/api/v1/auth`. JavaScript cannot read it, and the browser only sends it to the auth endpoints.
- After a page reload, the app calls `POST /auth/refresh`. The cookie is sent automatically and a new access token comes back. This is how the user stays logged in without the token ever being stored in JS-readable storage.
- **CSRF:** `SameSite=Strict` stops the cookie from being sent with cross-site requests. As defense in depth, `/auth/refresh` and `/auth/logout` also require a custom header (`X-Requested-With: bookly`). The CORS policy only allows the frontend origin, so another site cannot set that header.
- **Logout** revokes the refresh token on the server and clears the cookie.
- **Development:** the frontend uses the Vite dev-server proxy, so the API is same-origin and cookies work without cross-site settings.

### 4.3 Authorization
- Dashboard endpoints take the `business_id` **from the verified JWT only**, never from the URL, query string or body.
- A FastAPI dependency `get_current_user` provides the user. A second dependency, `require_role("owner")`, guards owner-only routes.
- Staff can only list bookings where `staff_id` equals their own staff record.

### 4.4 Tenant isolation
- Every tenant-owned table has a non-null `business_id`.
- All database access goes through **repository functions that require `business_id` as a parameter**. Route handlers never build unscoped queries.
- **Composite foreign keys** (for example `bookings(staff_id, business_id) → staff(id, business_id)`) make it impossible at the database level to link a booking to another business's staff member or service.
- Requests for another tenant's resources return **`404 Not Found`**, not `403`, so the API does not reveal that the resource exists.
- **Isolation test suite** (`backend/tests/integration/test_tenant_isolation.py`):
  - The test fixture creates Business A and Business B, each with an owner, a staff member, a service and a booking.
  - For **every** dashboard endpoint (list, get, create, update, delete, cancel), a parametrized test checks that A's token:
    - cannot read B's resources (B's IDs are missing from lists, and direct GETs return 404)
    - cannot update, delete or cancel B's resources (404, and B's data is unchanged afterwards)
    - cannot create resources that reference B's IDs (for example a booking with B's `staff_id`); these return 404 or 422
    - has any injected `business_id` in a request body ignored or rejected
  - Public endpoints:
    - A's slug never returns B's services or staff.
    - A cancel token only works on the booking it belongs to.
  - A **route-coverage test** lists every registered route and fails if a dashboard route is missing from the isolation test matrix. This stops new endpoints from skipping the checks.

### 4.5 Public endpoint protection
- **Rate limiting** uses `slowapi` and is keyed by client IP. Limits are configurable through environment variables.
  - `POST /public/.../bookings`: 5 per minute and 30 per hour per IP
  - `GET .../availability`: 60 per minute per IP
  - `POST /public/bookings/{token}/cancel`: 10 per minute per IP
  - `POST /auth/login`: 5 per minute per IP plus 20 per hour per email (brute-force protection)
  - When a limit is exceeded, the API returns `429 Too Many Requests` with a `Retry-After` header.
  - The client IP is read from `X-Forwarded-For` **only** when the request comes from a trusted proxy, which is configured per deployment.
  - The MVP stores rate-limit counters in memory. With more than one instance, they move to Redis.
- **Strict input validation** uses Pydantic v2 models with `extra="forbid"` and `strict=True` wherever possible.
  - `customer_name`: 1–100 chars, trimmed, control characters rejected
  - `customer_email`: `EmailStr`, max 254 chars
  - `customer_phone`: E.164 format (`^\+[1-9]\d{7,14}$`)
  - `starts_at`: timezone-aware ISO 8601, aligned to the 15-minute grid, within the lead time and horizon
  - IDs are UUIDs and slugs match `^[a-z0-9](?:[a-z0-9-]{1,48}[a-z0-9])$`
  - The request body size is capped at 16 KB.
  - A hidden **honeypot** form field must be empty. If it is filled in, the API returns a generic success response and stores nothing.
- **Cancel tokens** are 32 random bytes (`secrets.token_urlsafe`). Only their SHA-256 hash is stored, and they are compared in constant time.
- **Error responses** never include stack traces or SQL. They use one consistent shape: `{"error": {"code": "...", "message": "...", "details": [...]}}`.
- **Security headers:** CORS is restricted to the frontend origin, and responses set `X-Content-Type-Options`, `Referrer-Policy` and a basic CSP on the frontend.

---

## 5. Out of scope for the MVP

These items may appear in later phases, or not at all:

- Customer accounts and login
- Rescheduling (customers cancel and book again instead)
- Group classes and capacity-based bookings (for gyms)
- Payments, deposits and invoices
- Real email or SMS delivery, and reminders (the MVP logs them through the `EmailSender` interface)
- Recurring bookings
- Platform super-admin
- One user belonging to several businesses
- Calendar sync with Google or Outlook
- Custom booking-page branding and custom domains
- Internationalization (the UI is English only)
- Buffer times between appointments
- Waitlists, reviews and analytics dashboards

---

## 6. Data model

PostgreSQL 18. Primary keys are UUIDs (`uuid` type, generated in the app). Every table has `created_at` and `updated_at timestamptz`. Tables marked **(T)** are tenant-owned and have `business_id`.

```
businesses
  id               uuid PK
  name             varchar(100) not null
  slug             varchar(50)  not null unique
  timezone         varchar(64)  not null        -- IANA name, validated
  currency         char(3)      not null default 'EUR'

users (T)
  id               uuid PK
  business_id      uuid FK → businesses not null
  email            varchar(254) not null unique  -- case-insensitive (citext or lower() index)
  password_hash    varchar      not null
  role             enum('owner','staff') not null
  is_active        bool not null default true

refresh_tokens
  id               uuid PK
  user_id          uuid FK → users not null
  token_hash       char(64) not null unique
  family_id        uuid not null
  expires_at       timestamptz not null
  revoked_at       timestamptz null

staff (T)
  id               uuid PK
  business_id      uuid FK → businesses not null
  user_id          uuid FK → users null unique   -- set if the staff member can log in
  display_name     varchar(100) not null
  is_active        bool not null default true
  UNIQUE (id, business_id)                       -- target for composite FKs

services (T)
  id               uuid PK
  business_id      uuid FK → businesses not null
  name             varchar(100) not null
  duration_minutes int not null check (duration_minutes between 5 and 480 and duration_minutes % 5 = 0)
  price_cents      int not null check (price_cents >= 0)
  is_active        bool not null default true
  UNIQUE (id, business_id)

staff_services (T)
  business_id      uuid not null
  staff_id         uuid not null
  service_id       uuid not null
  PK (staff_id, service_id)
  FK (staff_id, business_id)   → staff(id, business_id)
  FK (service_id, business_id) → services(id, business_id)

working_hours (T)
  id               uuid PK
  business_id      uuid not null
  staff_id         uuid not null
  weekday          smallint not null check (weekday between 0 and 6)   -- 0 = Monday
  start_time       time not null
  end_time         time not null check (end_time > start_time)
  FK (staff_id, business_id) → staff(id, business_id)

time_off (T)
  id               uuid PK
  business_id      uuid not null
  staff_id         uuid not null
  starts_at        timestamptz not null
  ends_at          timestamptz not null check (ends_at > starts_at)
  reason           varchar(200) null
  FK (staff_id, business_id) → staff(id, business_id)

bookings (T)
  id                uuid PK
  business_id       uuid not null
  staff_id          uuid not null
  service_id        uuid not null
  customer_name     varchar(100) not null
  customer_email    varchar(254) not null
  customer_phone    varchar(16)  not null
  starts_at         timestamptz not null
  ends_at           timestamptz not null check (ends_at > starts_at)
  price_cents       int not null
  status            enum('confirmed','cancelled') not null default 'confirmed'
  source            enum('public','dashboard') not null
  cancel_token_hash char(64) not null unique
  cancelled_at      timestamptz null
  FK (staff_id, business_id)   → staff(id, business_id)
  FK (service_id, business_id) → services(id, business_id)
  EXCLUDE USING gist (staff_id WITH =, tstzrange(starts_at, ends_at) WITH &&)
          WHERE (status = 'confirmed')
  INDEX (business_id, starts_at)
```

Database migrations are managed with **Alembic**. The exclusion constraint needs the `btree_gist` extension, which is enabled in the first migration.

---

## 7. API overview

The base path is `/api/v1`. The API is JSON only. FastAPI generates OpenAPI docs at `/docs`.

### Health
| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/health` | — | Liveness check and database connectivity |

### Auth
| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/auth/register` | — | Create a business and its owner, then log in |
| POST | `/auth/login` | — | Return an access token and set the refresh cookie |
| POST | `/auth/refresh` | cookie | Rotate the refresh token and return a new access token |
| POST | `/auth/logout` | cookie | Revoke the refresh token and clear the cookie |
| GET | `/auth/me` | any | Current user, role and business |

### Dashboard (business taken from the token)
| Method | Path | Role | Description |
|---|---|---|---|
| GET / PATCH | `/business` | owner (PATCH) | View or update the business name, time zone and currency (the slug is immutable in the MVP) |
| GET / POST | `/services` | owner (POST) | List or create services |
| GET / PATCH / DELETE | `/services/{id}` | owner | Delete performs a soft deactivation if bookings exist |
| GET / POST | `/staff` | owner (POST) | List or create staff members |
| GET / PATCH / DELETE | `/staff/{id}` | owner | Manage a staff member |
| PUT | `/staff/{id}/services` | owner | Replace the set of services a staff member offers |
| GET / PUT | `/staff/{id}/working-hours` | owner (PUT) | Replace the weekly hours |
| GET / POST | `/staff/{id}/time-off` | owner (POST) | List or add time off |
| DELETE | `/staff/{id}/time-off/{time_off_id}` | owner | Remove time off |
| POST | `/staff/{id}/login` | owner | Create a login for a staff member |
| GET | `/bookings?from=&to=&staff_id=` | owner, staff (own only) | List bookings |
| POST | `/bookings` | owner | Create a booking manually |
| GET | `/bookings/{id}` | owner, staff (own only) | Booking details |
| POST | `/bookings/{id}/cancel` | owner | Cancel a booking |

### Public (rate-limited)
| Method | Path | Description |
|---|---|---|
| GET | `/public/businesses/{slug}` | Business name and time zone |
| GET | `/public/businesses/{slug}/services` | Active services |
| GET | `/public/businesses/{slug}/staff?service_id=` | Active staff who offer the service |
| GET | `/public/businesses/{slug}/availability?service_id=&date=&staff_id=` | Free slot start times for one local date |
| POST | `/public/businesses/{slug}/bookings` | Create a booking; returns the booking and the cancel token **once** |
| GET | `/public/bookings/{token}` | Booking summary |
| POST | `/public/bookings/{token}/cancel` | Cancel a booking |

### AI (phase 7)
| Method | Path | Description |
|---|---|---|
| POST | `/public/businesses/{slug}/assistant` | Natural-language booking assistant (see §9, phase 7) |

### Status codes
| Code | Meaning |
|---|---|
| `400` | Bad request |
| `401` | Missing or invalid token |
| `404` | Not found, or belongs to another tenant |
| `409` | Slot no longer available |
| `422` | Validation error |
| `429` | Rate limited |

---

## 8. Folder structure

The project is a monorepo:

```
bookly/
├── backend/
│   ├── app/
│   │   ├── main.py                 # FastAPI app factory, middleware, routers
│   │   ├── core/                   # config (pydantic-settings), security (JWT, hashing), rate_limit, errors
│   │   ├── db/                     # engine, session, Base
│   │   ├── models/                 # SQLAlchemy ORM models
│   │   ├── schemas/                # Pydantic request/response models
│   │   ├── repositories/           # tenant-scoped DB access (business_id required)
│   │   ├── services/               # business logic: availability, booking, notifications
│   │   └── api/
│   │       ├── deps.py             # get_db, get_current_user, require_role
│   │       └── v1/                 # routers: auth, business, services, staff, bookings, public
│   ├── alembic/                    # migrations
│   ├── tests/
│   │   ├── unit/                   # pure logic, e.g. availability calculation, DST edge cases
│   │   ├── integration/            # API + real Postgres, incl. test_tenant_isolation.py
│   │   └── conftest.py
│   ├── pyproject.toml              # deps (managed with uv), ruff, mypy, pytest config
│   └── .env.example
├── frontend/
│   ├── src/
│   │   ├── api/                    # typed API client, auth/refresh handling
│   │   ├── auth/                   # AuthProvider (in-memory token), protected routes
│   │   ├── features/
│   │   │   ├── dashboard/          # bookings, services, staff, settings pages
│   │   │   └── booking/            # public booking flow
│   │   ├── components/             # shared UI components
│   │   ├── lib/                    # date/time helpers, utils
│   │   ├── routes.tsx
│   │   └── main.tsx
│   ├── tests/                      # Vitest + React Testing Library (+ MSW for API mocks)
│   ├── package.json
│   └── vite.config.ts
├── docs/
│   └── SPEC.md
├── .github/workflows/              # ci.yml (phase 6)
├── docker-compose.yml              # Postgres for local dev (phase 1), full stack (phase 6)
├── CLAUDE.md
└── README.md
```

### Tooling decisions

**Backend**
- `uv` for dependencies
- `ruff` for linting and formatting
- `mypy` for type checking
- **SQLAlchemy 2.0, synchronous**, with `psycopg` 3. Sync is simpler to learn and fast enough for this scale.
- `pwdlib` (bcrypt) or `bcrypt` for password hashing (no `passlib`)
- `pytest` + `httpx` TestClient
- Integration tests run against **real PostgreSQL 18**, not SQLite, because the exclusion constraint and `timestamptz` behavior must be tested for real.

**Frontend**
- **React 19** + TypeScript in strict mode
- React Router
- TanStack Query for server state
- React Hook Form + Zod for forms and validation
- ESLint + Prettier
- Vitest + React Testing Library + MSW

---

## 9. Phases

Each phase ends with passing tests, a short demo-able result and a merge into `main` through a pull request.

### Phase 1 — Setup
- **Prerequisite:** Docker Desktop must be installed. It is not installed yet, and the developer will install it before this phase.
- Repository structure as described in §8, `.gitignore` (including `.env*` except `.env.example`) and `.editorconfig`.
- Backend skeleton: FastAPI app, `/api/v1/health`, settings from environment variables, ruff, mypy and pytest configuration.
- `docker-compose.yml` that runs only **PostgreSQL 18** for local development, plus a separate test database.
- Alembic initialized, with a first migration that enables `btree_gist`.
- Frontend skeleton: Vite React 19 + TypeScript app, ESLint, Prettier, Vitest and one smoke test.
- **Done when:** `pytest` and `npm test` both pass, and `/api/v1/health` returns OK with the database connected.

### Phase 2 — Backend MVP (no auth yet)
- Models and migrations for businesses, staff, services, staff_services, working_hours, time_off and bookings.
- Tenant-scoped repositories.
- An **availability service** that is pure and unit-tested. It covers every rule from R1 to R10, including DST transitions, breaks, time off and edge-touching bookings.
- A booking service covering R3, R5–R9, R11 and R13. It includes a concurrency test showing that two simultaneous inserts lead to exactly one success and one 409.
- Public endpoints with Pydantic validation, rate limiting, the honeypot field and cancel tokens.
- A temporary seed script that creates a demo business.
- **Done when:** a demo business can be booked and cancelled through `/docs`, and the unit and integration tests are green.

### Phase 3 — Auth and roles
- The users and refresh_tokens tables, password hashing with `pwdlib`/`bcrypt`, the register, login, refresh and logout flows, and the cookie handling described in §4.2.
- The `get_current_user` and `require_role` dependencies, plus all dashboard endpoints.
- Rate limiting on login.
- The **full tenant-isolation suite and the route-coverage test** (§4.4).
- **Done when:** the isolation suite covers 100% of the dashboard routes and passes.

### Phase 4 — React dashboard
- Signup and login pages, and an `AuthProvider` that keeps the token in memory and refreshes silently.
- Pages for bookings (day and week list), services CRUD, staff CRUD with working hours and time off, and business settings.
- Staff see a read-only view of their own schedule.
- Loading, empty and error states for every page, plus accessible forms (labels and keyboard navigation).
- Component tests with Vitest + RTL + MSW.

### Phase 5 — Public booking page
- `/b/:slug` with this flow: service → staff → date → slot → customer details → confirmation.
- A cancel page at `/b/:slug/booking/:token`.
- Times are shown in the business time zone, with the zone clearly labeled.
- Clear handling of 409 ("That slot was just taken, please pick another") and 429 responses.
- Mobile-first layout.

### Phase 6 — CI, Docker and deploy
- GitHub Actions on every pull request and push to `main`:
  - backend: ruff, mypy and pytest, with a PostgreSQL 18 service container
  - frontend: lint, type-check, Vitest and build
- Multi-stage Dockerfiles for the backend and frontend, with a `docker-compose.yml` that runs the full stack.
- Deploy to Render or Fly.io with managed Postgres. Secrets live only in the platform's secret store.
- Migrations run automatically on deploy.
- Dependabot for dependency updates.

### Phase 7 — AI booking
- An assistant on the public page. A customer can type something like "Haircut with Maria next Tuesday after 5pm".
- An LLM with **tool calling** uses existing services such as `list_services`, `find_availability` and `create_booking`. The LLM never writes to the database directly, and all the R1–R13 rules and validation still apply.
- The LLM always asks the customer to confirm before booking.
- The assistant is rate-limited, has a token/cost cap and is protected against prompt injection: tools are scoped to the business slug and the system prompt is not trusted for authorization.
- Tests mock the LLM client.

### Phase 8 — README
- The project pitch, a screenshot or GIF and a live demo link.
- An architecture diagram and the key design decisions: tenant isolation, the exclusion constraint and token storage.
- Local setup in 5 commands, how to run the tests and the CI badge.
- A "What I learned" section and a roadmap, which is the out-of-scope list from §5.
