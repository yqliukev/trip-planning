# API manual implementation

Saved from the service-layer planning chat. The to-do list is the last reply in that chat. The note below is the recommendation from [DTO recommendation](d931e070-29aa-47b1-86fc-6e3fbfbb1c35).

## To-do list for manual implementation of `api/`

The repo still has no application code, so this is a from-scratch build of `api/` against the current architecture. Do the shared-package and Compose work first; without them the API package has nowhere to import schemas from and no database to talk to.

### 0. Blocking work outside `api/`

Do these before writing any API source.

- [ ] Root `package.json` with npm workspaces: `shared`, `api` (`web` and `mcp` can be empty stubs until needed).
- [ ] `shared/` package: Zod schemas and inferred types for Trip and Location (create, update/partial, DTO). Export from `shared` only; `api` depends on it.
- [ ] DTO fields: `id` (UUID string), `title` / `name`, `notes`, trip `startDate` / `endDate` as `YYYY-MM-DD`, location `{ lat, lon }`, `sortOrder`, `createdAt` / `updatedAt` ISO UTC. List envelopes: `{ items: [...] }`.
- [ ] Root `.env.example` with at least `DATABASE_URL` and `API_PORT`.
- [ ] Root `docker-compose.yml` with `db` (PostGIS image, e.g. `postgis/postgis`) and later `api`. `web` can wait.

### 1. Package skeleton

- [ ] `api/package.json`: Node 22+, TypeScript, `hono`, `@hono/zod-openapi`, `@hono/node-server`, `zod`, `sequelize`, `pg`, `uuid`, `dotenv`. Dev: `typescript`, `tsx`, `sequelize-cli`, types.
- [ ] `api/tsconfig.json`: ESM, `strict`, path/workspace resolution to `shared`.
- [ ] Scripts: `dev` (`tsx watch`), `build`, `start`, `db:migrate`, `db:migrate:undo`.
- [ ] `api/Dockerfile` (Node 22, copy `shared` + `api`, run migrations then start). Wire it into Compose as `api`, depends on `db`.
- [ ] `api/.sequelizerc` pointing at `src/db/config.cjs` (or equivalent) and `migrations/`.

Suggested layout:

```
api/
  package.json
  tsconfig.json
  Dockerfile
  .sequelizerc
  src/
    index.ts              # listen
    app.ts                # Hono + OpenAPI app
    config.ts             # env
    errors.ts             # NotFound / Validation / Conflict
    db/sequelize.ts
    db/models/Trip.ts
    db/models/Location.ts
    services/trips.ts
    services/locations.ts
    routes/trips.ts
    routes/locations.ts
    middleware/error.ts
  migrations/
```

### 2. Config and Sequelize

- [ ] `config.ts`: read `DATABASE_URL`, `PORT`, `CORS_ORIGIN` (Vite origin). Fail fast if `DATABASE_URL` is missing.
- [ ] Sequelize init from `DATABASE_URL`. No models query the DB until `authenticate()` succeeds on boot.
- [ ] First migration: `CREATE EXTENSION IF NOT EXISTS postgis;`, then `trips` and `locations`.
- [ ] Columns: UUID PKs (no DB default — the service generates them), `locations.trip_id` FK with `ON DELETE CASCADE`, `geometry(Point, 4326)` for the point, `sort_order` integer, timestamps.
- [ ] Sequelize models matching that schema. Point stored as geometry; conversion to `{ lat, lon }` stays in the service, not in the model getters if you can help it (one place only).

### 3. Errors and service contract

Nothing under `src/services/` imports Hono.

- [ ] Typed errors: `NotFoundError`, `ValidationError`, `ConflictError`.
- [ ] Every service function: one validated input object + optional `{ transaction }`. Parse the `shared` Zod schema **inside** the function (MCP will skip HTTP validation later).
- [ ] Return plain DTOs, never Sequelize instances.
- [ ] UUID v4 generated in the service on create.
- [ ] Trip dates stay date-only strings; timestamps ISO UTC.

Trip functions:

- [ ] `listTrips` → `{ items }`
- [ ] `getTrip`
- [ ] `createTrip`
- [ ] `updateTrip` (partial)
- [ ] `deleteTrip` (locations go via CASCADE)

Location functions:

- [ ] `listLocations` (by `tripId`, order by `sortOrder`, then `id`)
- [ ] `getLocation`
- [ ] `createLocation` — 404 if trip missing; `sortOrder = max(trip) + 1`; convert `{ lat, lon }` → `ST_SetSRID(ST_MakePoint(lon, lat), 4326)`
- [ ] `updateLocation` (partial, including optional point)
- [ ] `deleteLocation`
- [ ] `reorderLocations` — rewrite the whole list for one trip in a single transaction; 400 if the id set doesn’t match the trip’s locations

### 4. HTTP layer

Thin handlers: validate path/body with `@hono/zod-openapi`, call one service function, return the DTO.

- [ ] CORS open to the Vite origin.
- [ ] Error middleware: `NotFoundError` → 404, `ValidationError` → 400, `ConflictError` → 409, everything else → 500. No stack traces in responses.
- [ ] `GET /health` (Compose needs it).
- [ ] OpenAPI doc route (`/openapi.json` and/or `/doc`).

Routes:

| Method | Path | Service |
|---|---|---|
| GET | `/trips` | `listTrips` |
| POST | `/trips` | `createTrip` |
| GET | `/trips/:id` | `getTrip` |
| PATCH | `/trips/:id` | `updateTrip` |
| DELETE | `/trips/:id` | `deleteTrip` |
| GET | `/trips/:id/locations` | `listLocations` |
| POST | `/trips/:id/locations` | `createLocation` |
| GET | `/trips/:id/locations/:locationId` | `getLocation` |
| PATCH | `/trips/:id/locations/:locationId` | `updateLocation` |
| DELETE | `/trips/:id/locations/:locationId` | `deleteLocation` |
| PUT | `/trips/:id/locations/order` | `reorderLocations` |

Keep location writes nested under the trip so the URL matches the FK. Do not add a combined upsert.

### 5. Boot and check

- [ ] `src/index.ts`: load config, init Sequelize, mount app, listen.
- [ ] `compose up db`, run migrations, `npm run dev -w api`.
- [ ] Hit `/health`, create a trip, add two locations, reorder, delete the trip and confirm locations are gone.
- [ ] Confirm OpenAPI lists every route above and that request/response schemas come from `shared`, not duplicated in the route files.

### Out of scope for this pass

Auth, MCP, `web/`, routing/commute, pagination, tests (framework still deferred), anything beyond Trip and Location. If a handler grows a query or a business rule, it belongs in the service instead.

## Note: DTO recommendation

Source: assistant reply in `agent-transcripts/d931e070-29aa-47b1-86fc-6e3fbfbb1c35/d931e070-29aa-47b1-86fc-6e3fbfbb1c35.jsonl` (2026-09-15). A canvas of the same review lives at `canvases/dto-recommendation.canvas.tsx` in the Cursor project folder.

**Use DTOs** — as Zod schemas in `shared/` plus a small mapper in the service, not as a class/mapper layer.

The service-layer plan already defined this: every service function returns a plain JSON object, never a Sequelize instance. HTTP and MCP both emit that object. Create, update, and DTO are three Zod schemas per resource. That is the right grain for this repo.

**Why it should be used.** The architecture already requires a wire type whether you name it DTO or not. Handlers and MCP tools share one service layer and must return the same payload; Sequelize instances do not serialize stably. `web` cannot import models from `api`, so the frontend type has to live in `shared/` as a Zod-inferred object. PostGIS `geometry(Point, 4326)` is not a useful JSON value — callers need `{ lat, lon }`, and only the service should know storage is lon-first. OpenAPI is generated from Zod, so documenting one shape and returning another would be a bug.

**The real costs** are mapping boilerplate (small: two `toXDto` functions) and schema drift between DB, model, and Zod. The failure mode is over-building: list vs detail DTOs, class-transformer, extra interfaces that duplicate `z.infer`. Re-parsing the output schema on every response is not worth it for a single-user v1.

**Skip these alternatives:** `model.toJSON()`, Sequelize getters for `{ lat, lon }`, and DTO classes on top of Zod. Each still needs a mapping step and fights the `shared/` + OpenAPI path.

Implement it as `tripDto` / `locationDto` in `shared/`, `z.infer<typeof tripDto>` as the TypeScript type, and `toTripDto` / `toLocationDto` called only from services. Keep create and update as separate input schemas. Wrap lists as `{ items: T[] }`. Do not split payloads until they actually differ.
