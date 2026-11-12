# Sweety - Blood Glucose Tracker

Sweety is a small full-stack app for recording personal blood glucose readings and viewing daily and monthly summaries. The API is built with Node.js, Express, and PostgreSQL; the client uses React and Vite.

## Requirements

- Node.js 20 or newer and npm
- PostgreSQL 13 or newer

## Installation and database setup

1. Install the project dependencies from the repository root:

   ```sh
   npm install
   ```

2. Create a PostgreSQL database named `sweety` (or choose another name):

   ```sql
   CREATE DATABASE sweety;
   ```

3. Copy `.env.example` to `.env` in the repository root and set `DATABASE_URL` and `JWT_SECRET`. Use a randomly generated secret of at least 32 characters. For a local PostgreSQL install, the example URL may be adjusted to match your local username and password.

4. Apply the schema:

   ```sh
   psql "postgresql://postgres:postgres@localhost:5432/sweety" -f backend/db/schema.sql
   ```

   The schema creates users and readings, case-insensitive unique emails, foreign keys, valid-slot and value constraints, and the per-user/per-day/per-slot uniqueness constraint.

## Run the application

Start the API and React development server together from the repository root:

```sh
npm run dev
```

The client is available at <http://localhost:5173> and the API at <http://localhost:4000>. Vite proxies `/api` requests to the API. To run either service separately:

```sh
npm run dev --workspace backend
npm run dev --workspace frontend
```

For a frontend production build:

```sh
npm run build
```

In production, Express serves the built React frontend and API on the same origin, so the HTTP-only session cookie works without a separate frontend host.

## Deploy to Render

The included `render.yaml` Blueprint provisions a web service and PostgreSQL database. The web service builds the React app, runs the idempotent database schema migration before starting, and serves the frontend and API from one origin. Render automatically supplies a strong `JWT_SECRET` and the database connection URL.

1. Push this project to a GitHub repository you control. Do not commit `.env`; it is ignored by Git.
2. In Render, create a **Blueprint** and connect that repository. Approve the services described in `render.yaml`.
3. Wait for the database and web service deployment to finish, then open the Render web service URL.
4. The Blueprint uses Render's free plans where available. Free web services may sleep when idle, and free database availability, lifetime, and limits depend on Render's current offering. For an always-on app and persistent production data, select suitable paid plans in Render before creating the services.

The app's `/api/health` endpoint is used for deployment health checks. Never use the seeded demo password or sample readings as production credentials or patient data.

## Demo data

After applying the schema and setting up `.env`, seed a demo account and readings:

```sh
npm run seed
```

Demo login:

- Email: `demo@sweety.local`
- Password: `SweetyDemo123!`

The seed script is repeatable. It refreshes the demo account password and adds missing sample rows without duplicating existing sample readings.

## Tests

Run the backend validation tests:

```sh
npm test
```

They use Node's built-in test runner and do not require a running database.

## API overview

- `POST /api/auth/register`, `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me`
- `GET /api/readings`, `POST /api/readings`, `PUT /api/readings/:id`, `DELETE /api/readings/:id`
- `GET /api/reports?type=daily&date=YYYY-MM-DD`
- `GET /api/reports?type=month-to-date&date=YYYY-MM-DD`
- `GET /api/reports?type=previous-month`
- `GET /api/reports?type=all-time`
- `GET /api/health`

Auth routes set an HTTP-only JWT cookie. Reading and report routes always scope database queries to the authenticated user. Request bodies are validated in the API, and the database enforces reading value, slot, and uniqueness constraints as a second line of defense.

## Environment variables

| Variable | Required | Description |
| --- | --- | --- |
| `NODE_ENV` | No | Runtime mode; use `production` to enable the cookie's `Secure` attribute. |
| `PORT` | No | Express API port. Defaults to `4000`. |
| `CLIENT_ORIGIN` | No | Exact browser origin allowed by CORS. Defaults to `http://localhost:5173`. |
| `DATABASE_URL` | Yes | PostgreSQL connection URL. |
| `DATABASE_SSL` | No | Set to `true` to require verified TLS for PostgreSQL. Defaults to `false`. |
| `JWT_SECRET` | Yes | Private signing secret of at least 32 characters. |

## Notes and assumptions

- Calendar dates and report periods use UTC consistently between the API and browser date inputs.
- A glucose value is an integer in mg/dL; accepted values are 40 through 600, inclusive.
- A user may have at most one reading for each of the four allowed time slots on a date. The unique database constraint prevents duplicate slot readings; the API also returns the specified daily-limit message when all four readings already exist.
- Month-to-date reports require a selected date in the current UTC month and include the first of that month through the selected date, inclusive. The previous-month report covers the complete prior UTC calendar month.
- The all-time report provides the overall summary. Daily reports include each reading and its time slot. Aggregate reports include highest, lowest, one-decimal average, and reading count; an empty period returns a zero count and null statistics.
- This app stores and summarizes measurements; it does not interpret readings or provide medical advice.
