# Unit & Integration Testing Plan — Lifeline Alert

## Context
The team is setting up a full test suite for "Lifeline Alert", a Next.js 16 blood donation coordination platform. There are currently **zero tests** in the codebase. This plan covers Unit and Integration testing only (not E2E, Compatibility, Responsive, or UAT). We are using Jest.

The application has 3 user roles (Donor, Hospital, Admin), ~12 API routes, ~13 Prisma models, and several utility/library functions that are already prime candidates for testing.

---

## Recommended Organisation: By Layer, Not By Persona

**Personas are the right lens for E2E and UAT** — those tests simulate a real user journey ("as a donor, I can accept an alert"). For unit and integration testing, organising **by layer** is better because:
- Unit tests target isolated functions with no awareness of "who is using" them
- Integration tests target API routes — most routes already enforce role via session, so the role is part of the test input, not the organising principle
- Persona-based splits create awkward overlaps (e.g. the auth route is used by all 3 roles)

---

## Team Split: 2 Members is Right

The testable surface is well-defined. Here is a realistic scope breakdown:

| Area | Estimated Tests | Difficulty | Assignee |
|------|----------------|------------|----------|
| **Member A** — Unit tests (pure functions) | ~30-40 test cases | Low | 1 person |
| **Member B** — Integration tests (API routes) | ~40-60 test cases | Medium-High | 1 person |

4 members would result in very thin slices with a lot of coordination overhead. 2 is the right call. A 3rd member could optionally take on **component unit tests** (React Testing Library for forms) if scope allows.

---

## Phase 1: Setup (Both Members, Day 1)

Both members should do this together once, then split.

### Install dependencies
```bash
npm install --save-dev jest jest-environment-jsdom @testing-library/react @testing-library/jest-dom ts-jest @types/jest
```

### Files to create
- `jest.config.ts` — root Jest config (use `next/jest` transformer)
- `jest.setup.ts` — global setup (import `@testing-library/jest-dom`)
- `src/__tests__/` — top-level test directory
  - `unit/` — pure function tests
  - `integration/` — API route tests

---

## Member A — Unit Tests (Pure Functions)

All targets are in `src/lib/`. These need **no mocking** and can be written very fast.

### Target files and functions

**`src/lib/periodConversion.ts`**
- `toHours(period)` — test all 4 valid inputs ("24h", "72h", "1w", "1m") and invalid inputs
- `toPeriod(hours)` — test all valid hour values and edge cases

**`src/lib/alertSettingsValidation.ts`** (highest value — complex logic)
- `validateMaxValue(value, fieldName)` — test boundary values (0, 1, 10000, 10001, negative, non-integer)
- `validatePeriod(period)` — test all valid periods and invalid strings
- `validateRadius(value, fieldName)` — test boundaries (0, 1, 20000, 20001)
- `validateBoolean(value, fieldName)` — test true/false/null/string
- `validatePeriodOrdering(newHours, newMax, existingRules)` — most complex; test ordering conflicts, no-conflict cases, empty rules

**`src/lib/displayMaps.ts`**
- Verify all 8 blood types map to correct display strings
- Verify all urgency levels (LOW/MEDIUM/HIGH/CRITICAL) map correctly
- Verify all alert statuses (PENDING/APPROVED/REJECTED) map correctly

**`src/lib/notifications.ts`** (mock Prisma)
- `isBloodTypeMatch(donor, alert)` — test all compatible/incompatible type combos
- `isDonorEligible(donor, alert, organisation)` — test combined eligibility logic

**`src/app/api/auth/register/route.ts`** — extract `calculateAge()` first, then test:
- `calculateAge(dob)` — test typical ages, boundary ages (17, 60, 70), future dates, null

### Test file locations
```
src/__tests__/unit/periodConversion.test.ts
src/__tests__/unit/alertSettingsValidation.test.ts
src/__tests__/unit/displayMaps.test.ts
src/__tests__/unit/notifications.test.ts
src/__tests__/unit/calculateAge.test.ts
```

---

## Member B — Integration Tests (API Routes)

Integration tests call the actual Next.js route handlers with a **test database** (or mocked Prisma client). The recommended approach for Next.js App Router is to import the route handler functions directly and call them with mock `Request` objects — no running server needed.

### Prisma mocking strategy
Use `jest.mock()` to mock `src/lib/prisma.ts` and control what the DB returns per test. This avoids needing a live database and makes tests fast and deterministic.

```ts
jest.mock('@/lib/prisma', () => ({ prisma: mockDeep<PrismaClient>() }))
```

### Target API routes (priority order)

**Auth routes** (`src/app/api/auth/`)
- `POST /api/auth/register` — valid donor registration, missing fields, invalid age (<17, >70), invalid postcode, weak password, duplicate email
- `POST /api/auth/register-hospital` — valid hospital, missing NHS ID, invalid email

**Admin — Alert Management** (`src/app/api/admin/alerts/`)
- `GET /api/admin/alerts` — returns list, requires admin session
- `PUT /api/admin/alerts/[id]` — approve/reject alert, non-admin gets 403
- `DELETE /api/admin/alerts/[id]` — delete alert, non-admin gets 403

**Hospital — Alerts** (`src/app/api/hospital/alerts/`)
- `POST /api/hospital/alerts` — create alert with valid data, unauthenticated gets 401
- `GET /api/hospital/alerts` — returns only the requesting hospital's alerts

**Hospital — Stock** (`src/app/api/hospital/stock/`)
- `GET /api/hospital/stock` — returns stock for authenticated hospital
- `POST /api/hospital/stock` — update stock levels

**Admin — Hospitals** (`src/app/api/admin/hospitals/`)
- `GET /api/admin/hospitals` — pagination and search work correctly
- `PUT /api/admin/hospitals/[id]` — update hospital info

**Notifications** (`src/app/api/notifications/`)
- `GET /api/notifications` — returns only donor's own notifications
- `PUT /api/notifications/read` — marks as read, verifies correct donor session

### Role/auth testing pattern
Every protected route should have at least:
1. A test with the correct role session → `200 OK`
2. A test with no session → `401 Unauthorized`
3. A test with the wrong role session → `403 Forbidden`

### Test file locations
```
src/__tests__/integration/auth.register.test.ts
src/__tests__/integration/auth.register-hospital.test.ts
src/__tests__/integration/admin.alerts.test.ts
src/__tests__/integration/hospital.alerts.test.ts
src/__tests__/integration/hospital.stock.test.ts
src/__tests__/integration/admin.hospitals.test.ts
src/__tests__/integration/notifications.test.ts
```

---

## Critical Files to Read Before Starting

| File | Why |
|------|-----|
| `src/lib/alertSettingsValidation.ts` | Primary unit test target — understand the validation logic |
| `src/lib/notifications.ts` | Unit test target — understand eligibility logic |
| `src/lib/periodConversion.ts` | Unit test target |
| `src/lib/authOptions.ts` | Needed to understand how to mock sessions in integration tests |
| `src/app/api/auth/register/route.ts` | Integration test target + `calculateAge` extraction |
| `src/app/api/hospital/alerts/route.ts` | Integration test target |
| `src/app/api/admin/alerts/route.ts` | Integration test target |
| `prisma/schema.prisma` | Needed to construct correct mock data shapes |

---

## Verification

After writing tests:
1. Run `npx jest --coverage` — aim for >70% coverage on `src/lib/` and `src/app/api/`
2. All tests should pass with no live DB required (mocked Prisma)
3. Check that every protected API route has a 401/403 test case
4. Run `npx jest --testPathPattern=unit` and `npx jest --testPathPattern=integration` separately to confirm the split works
