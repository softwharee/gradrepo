# Technical Design – Routeflow

---

## Introduction

This document describes the technical design for **Routeflow**, a web application that automates daily route planning for small transport companies. It serves as the technical foundation for implementation, covering system architecture, technology choices, security, geocoding, frontend architecture, API design, and scalability considerations.

This design is directly based on the **Functional Design – Routeflow**, which describes the intended behaviour, domain model, wireframes, and business rules in detail. Where relevant, sections in this document refer back to specific parts of the functional design.

---

## Functional Summary

Routeflow allows a planner (typically a business owner or senior driver) to:

- Manage a fleet of trucks and their properties (capacity, weight, dimensions, home depot).
- Enter transport orders per day (customer, address, pallet count, weight, time window).
- Generate an optimised daily route plan in one click, using a VRP algorithm backed by real driving distances from OSRM.
- Review the result on an interactive map, manually reorder stops via drag-and-drop, and finalise the plan.

The system supports a single user role (planner), authenticates via a passwordless email code flow, and operates in the Europe/Amsterdam timezone. The target scale is 5–15 trucks and 30–60 orders per day.

Constraints enforced by the system (from the Functional Design — Constraints and Business Rules):
- Pallet capacity must not be exceeded at any point in a route.
- Delivery time windows (all day / morning / afternoon / custom) must be respected.
- EU maximum driving time of 9 hours per day (Regulation (EC) No 561/2006).
- Every route starts and ends at the home depot of the assigned truck.

---

## Technical Overview — C4 Level 2 (Container Diagram)

The diagram below shows the Routeflow system at container level. Each box is a separately deployable unit or external system.

```
╔══════════════════════════════════════════════════════════════════════╗
║                         Routeflow System                            ║
║                                                                      ║
║  ┌─────────────────────┐   HTTPS/JSON    ┌────────────────────────┐ ║
║  │                     │◄───────────────►│                        │ ║
║  │    React SPA        │                 │    Go API Server       │ ║
║  │  (Vite, Leaflet,    │                 │  (REST API, VRP solver,│ ║
║  │   dnd-kit,          │                 │   auth, OSRM client,   │ ║
║  │   TanStack Query)   │                 │   geocoding proxy)     │ ║
║  │                     │                 │                        │ ║
║  └─────────────────────┘                 └───────────┬────────────┘ ║
║         │ served by                                  │             ║
║  ┌──────▼──────────────┐              ┌──────────────▼───────────┐  ║
║  │   Nginx             │              │      PostgreSQL 16        │  ║
║  │  (static file serve)│              │  (all domain entities,   │  ║
║  └─────────────────────┘              │   OTP codes, locations)  │  ║
║                                       └──────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════╝
          │                          │                    │
          ▼                          ▼                    ▼
 ┌─────────────────┐      ┌────────────────────┐  ┌─────────────────┐
 │  Postmark       │      │  OSRM              │  │  Nominatim      │
 │  (email)        │      │  (routing engine,  │  │  (geocoding,    │
 │                 │      │   self-hosted)     │  │   self-hosted)  │
 │  OTP delivery   │      │  /table endpoint   │  │  address→lat/lng│
 └─────────────────┘      └────────────────────┘  └─────────────────┘
```

### Container descriptions

| Container | Technology | Responsibility |
|-----------|-----------|----------------|
| React SPA | React 18, Vite, Leaflet.js, dnd-kit, TanStack Query | All UI: map, route cards, forms, drag-and-drop stop reordering |
| Go API Server | Go 1.22, chi router, pgx | REST API, OTP auth, VRP solver, OSRM calls, geocoding proxy |
| PostgreSQL 16 | PostgreSQL | Persistent storage for all domain entities and OTP state |
| Nginx | Nginx | Serves the compiled React SPA static files |
| Caddy | Caddy | TLS termination, reverse proxy to Nginx and Go API |
| OSRM | OSRM backend (Docker) | Driving distance and duration matrix for the Netherlands/Europe |
| Nominatim | Nominatim (Docker) | Geocoding: converts addresses entered by the planner to lat/lng coordinates |
| Postmark | Postmark (SaaS) | Transactional email for OTP code delivery |

---

## Deployment Architecture

All containers run on a single VPS (e.g. Hetzner CX32 or OVH VPS) managed via Docker Compose.

```
Internet (HTTPS :443)
        │
        ▼
┌───────────────────┐
│   Caddy           │  ← TLS termination (Let's Encrypt auto-cert)
│   reverse proxy   │
└──────┬────────────┘
       │
       ├─ /api/*  ──────────────► Go API (container port 8080)
       │                               │
       │                               ├──► PostgreSQL (port 5432, internal only)
       │                               ├──► OSRM       (port 5000, internal only)
       │                               └──► Nominatim  (port 8080, internal only)
       │
       └─ /*  ──────────────────► Nginx (serves React SPA, container port 3000)
```

**Docker Compose services:**

| Service | Image | Exposed externally |
|---------|-------|--------------------|
| caddy | caddy:2-alpine | :80, :443 |
| api | routeflow/api (custom Go build) | No (via Caddy) |
| web | routeflow/web (Nginx + React build) | No (via Caddy) |
| postgres | postgres:16-alpine | No |
| osrm | osrm/osrm-backend | No |
| nominatim | mediagis/nominatim | No |

PostgreSQL data is persisted via a Docker named volume. OSRM uses a pre-processed OSM extract for the Netherlands (`.osrm` files), mounted as a read-only volume.

---

## Frontend Architecture

### Technology stack

| Library | Purpose |
|---------|---------|
| React 18 | Component model, rendering |
| Vite | Build tool, dev server |
| React Router v6 | Client-side routing between screens |
| TanStack Query | Server state: fetching, caching, refetching |
| Leaflet + react-leaflet | Interactive map with coloured pins and route lines |
| dnd-kit | Drag-and-drop reordering of stops within a route |
| Zod | Client-side form validation schema |

### Routing

React Router v6 handles navigation between screens. All routes are protected by an `<AuthGuard>` component that redirects unauthenticated users to `/login`.

```
/login               → LoginPage
/verify              → VerifyPage
/dashboard           → DashboardPage       [protected]
/trucks              → TrucksPage          [protected]
/orders              → OrdersPage          [protected]
/routes              → RoutesPage          [protected]
/settings            → SettingsPage        [protected]
```

### State management

Global authentication state (user object, JWT presence) is held in a React Context (`AuthContext`). All server data (trucks, orders, plans) is managed by TanStack Query, which handles caching, background refetching, and loading/error states. No additional global state library (Redux, Zustand) is needed.

### Drag-and-drop stop reordering

The functional design specifies that the planner can drag stops within a truck's route to manually adjust the sequence (Screen 6 — Routes). This is implemented with **dnd-kit**:

1. Each stop within a route card is wrapped in a `<SortableItem>` component provided by dnd-kit's `@dnd-kit/sortable` package.
2. Stops are contained in a `<SortableContext>` per truck route.
3. On drag end, the updated stop order is sent to `PUT /api/plans/{date}/routes/{routeId}/stops` with the new sequence array.
4. TanStack Query invalidates the plan cache, causing the map to re-render with updated route lines.
5. Drag handles (the `≡` icon from the functional design) are the only drag trigger — the rest of the stop card is not draggable, preventing accidental reorders.

Cross-route dragging (moving a stop from one truck to another) is explicitly **not** supported in this version, consistent with the functional design.

### Map (Leaflet)

- Each truck has a colour stored in the database. Pins use that colour via Leaflet's `divIcon` with an inline-styled SVG marker.
- Numbered pins (1, 2, 3...) correspond to stop sequence within each truck's route.
- Route lines are drawn as `<Polyline>` components connecting stops in sequence, including the depot as start and end point.
- When a stop is manually reordered, the polyline updates immediately via a local optimistic update (`TanStack Query setQueryData`), giving instant visual feedback without waiting for the server round-trip.

### Geocoding in the frontend

When the planner types an address in the order form (Screen 5 — Orders), the address is geocoded to lat/lng before saving. Implementation:

1. The address input triggers a debounced call (300ms) to `GET /api/geocode?q={address}` after the user stops typing.
2. The API proxies the request to Nominatim and returns the top results as `[{ lat, lng, display_name }]`.
3. A small suggestion dropdown shows matched addresses for the planner to confirm.
4. On confirmation, lat/lng is stored alongside the order in the `Location` table.

If Nominatim returns no results, the frontend shows: "Address not found. Please check the address." A manual lat/lng fallback input is shown.

---

## Design Choices

### 1. Go for the backend

**Choice:** The API server is implemented in Go (1.22) using the `chi` router.

**Motivation:** Go compiles to a single static binary with no runtime dependencies, making deployment straightforward. Its standard library covers HTTP, JSON encoding, and PostgreSQL access (via `pgx`). Native goroutines make parallel OSRM requests during distance matrix computation trivial. `chi` is a lightweight, idiomatic HTTP router that adds middleware support (logging, CORS, rate limiting) without the overhead of a full framework like Gin.

**Alternatives considered:**

- **Node.js / TypeScript**: Familiar and fast to prototype, but async complexity grows quickly and runtime errors are harder to catch without strict discipline.
- **Python / FastAPI**: Excellent for data tasks but limited by the GIL for concurrent computation. The VRP solver benefits from Go's native concurrency.

---

### 2. React + Vite for the frontend

**Choice:** React 18 with Vite as the build tool.

**Motivation:** React has strong tooling and Leaflet.js has reliable React bindings (`react-leaflet`). Vite provides a fast development server and optimised production bundles. TanStack Query handles server state, reducing manual loading/error state management throughout the app.

**Alternatives considered:**

- **Next.js**: Adds server-side rendering, which is unnecessary here. The app is fully behind authentication and does not need SEO.
- **Vue + Vite**: Also viable. React was chosen based on familiarity and ecosystem depth for the required libraries (react-leaflet, dnd-kit).

---

### 3. PostgreSQL for persistence

**Choice:** PostgreSQL 16.

**Motivation:** The domain model from the functional design has a clear relational structure (plans → routes → stops → orders, with multiple foreign keys). PostgreSQL enforces referential integrity at the database level, supports UUID primary keys natively (`gen_random_uuid()`), and provides timezone-aware timestamps (`TIMESTAMPTZ`) — essential since the functional design specifies all times are in Europe/Amsterdam. The `pgx` driver for Go provides a high-performance connection pool.

**Alternatives considered:**

- **SQLite**: Not suitable for a hosted multi-user web application. No concurrent write support.
- **MongoDB**: The document model does not map naturally to the relational structure. Foreign key constraints and joins are significantly more natural in SQL for this domain.

---

### 4. Passwordless authentication (OTP via email)

**Choice:** A 6-digit one-time code sent to the user's email address, expiring after 10 minutes.

**Motivation:** The functional design specifies passwordless login. Planners use the app daily; passwords add unnecessary friction. OTP codes are simple to implement securely. Codes are stored as bcrypt hashes and invalidated on first use.

**Implementation flow:**
1. User submits email → server generates 6-digit code, stores `bcrypt(code)` + expiry + `user_id` in `otp_codes` table.
2. Server sends code via Postmark.
3. User submits code → server checks hash, verifies expiry, issues signed JWT as HttpOnly cookie, marks code as used.
4. All subsequent requests authenticate via the JWT.

**Security details:**
- The server always responds identically whether the email exists or not (no user enumeration).
- Rate limit: 5 OTP requests per email address per 15 minutes.
- 6 digits = 1,000,000 possibilities. With a lockout after 3 failed attempts, brute force is not practical within the 10-minute window.

**Alternatives considered:**

- **Magic link**: Equivalent security. OTP was chosen to keep the user in the same browser tab without depending on clicking a link from a mail client.
- **OAuth (Google/Microsoft)**: Adds a dependency on a third-party identity provider. Not appropriate for a small B2B tool.

---

### 5. JWT via HttpOnly cookie

**Choice:** After OTP verification, the server issues a signed JWT stored in an HttpOnly, Secure, SameSite=Strict cookie. Expiry: 7 days.

**Motivation:** HttpOnly cookies are inaccessible to JavaScript, eliminating XSS-based token theft. SameSite=Strict prevents CSRF. The token is signed with HS256 using a server-side secret. It contains the user's UUID and expiry. The server validates it on every request without a database lookup.

**Alternatives considered:**

- **localStorage**: Simpler but vulnerable to XSS. Rejected on security grounds.
- **Server-side session table**: Allows revocation but adds a database read per request. JWT expiry (7 days) with short OTP codes provides adequate security without per-request database overhead.

---

### 6. VRP solver — greedy insertion + 2-opt, embedded in Go

**Choice:** A custom VRP solver embedded in the Go API server.

**Motivation:** The target problem size (5–15 trucks, 30–60 orders) is small. A greedy nearest-neighbour insertion heuristic followed by 2-opt local search produces plans within a few percent of optimal and runs in well under one second. Embedding the solver avoids an additional service and simplifies deployment.

**Algorithm in detail:**

```
Input:
  - Distance/duration matrix (N×N) from OSRM
  - Orders: location, pallet_count, type, time_window
  - Trucks: capacity_pallets, home_depot, status=available

Step 1 — Build distance matrix
  Call OSRM /table with all unique locations (depots + order locations)
  Store as float64 matrix indexed by location ID

Step 2 — Greedy insertion
  Sort orders by time window tightness (tightest window first)
  For each unassigned order:
    Find truck + insertion position with lowest marginal distance cost
    that satisfies: capacity constraint, time window, 9h driving limit
    Assign order to that truck at that position

Step 3 — 2-opt improvement (per route)
  For each pair of stops (i, j):
    Compute cost of reversing the sub-sequence between i and j
    If reversal reduces total route distance: apply it, restart
  Repeat until no improvement found or 500ms wall-clock limit reached

Step 4 — Calculate arrival times
  For each route, walk stops in sequence:
    expected_arrival[stop] = departure_time + cumulative travel time
  Check time window violations (warn, shown to planner)

Output:
  Plan with routes, stops, arrival times, pallet deltas, totals
```

**Constraints enforced (from Functional Design — Constraints and Business Rules):**
- Pallet count tracked at every stop: never exceeds `capacity_pallets`.
- Time windows (morning 08:00–12:00, afternoon 12:00–17:00, custom) respected during insertion.
- Maximum 9 hours driving time per route (EU Regulation EC 561/2006).
- Every route starts and ends at the truck's home depot.

**Alternatives considered:**

- **OR-Tools (Google)**: Industry-standard solver. Requires a separate Python/C++ process (Docker sidecar or gRPC), significantly increasing deployment complexity. Not justified at this problem scale.
- **External SaaS (e.g. Routific)**: Paid API, adds a network dependency to the critical planning path, and exposes customer addresses to a third party.

---

### 7. OSRM for routing

**Choice:** OSRM (Open Source Routing Machine), self-hosted via Docker.

**Motivation:** Specified in the functional design. OSRM provides a `/table` endpoint returning an N×N matrix of driving durations and distances in a single HTTP call — essential for the VRP solver.

**OSRM `/table` call:**

```
GET http://osrm:5000/table/v1/driving/{lon,lat;lon,lat;...}
    ?sources=all&destinations=all&annotations=duration,distance

Response:
{
  "durations": [[0, 847, ...], [823, 0, ...], ...],  // seconds
  "distances": [[0, 12400, ...], [12380, 0, ...], ...]  // metres
}
```

The Go API extracts both matrices and passes them to the VRP solver. Because OSRM uses real road networks, the solver respects actual driving times rather than straight-line distances.

**Self-hosting setup:**
1. Download Netherlands OSM extract (Geofabrik).
2. Pre-process: `osrm-extract`, `osrm-partition`, `osrm-customize` (MLD pipeline).
3. Run `osrm-routed` on the processed files.
4. Disk: ~3 GB for the Netherlands; ~30 GB for full Europe.

**Alternatives considered:**

- **Public OSRM demo server**: Zero setup, development only. Rate-limited.
- **OpenRouteService**: Supports truck-specific routing (weight/height limits). Could replace OSRM in a future version since the domain model already stores `length_m`, `width_m`, `height_m`.
- **Google Maps Distance Matrix API**: Accurate but paid and a hard external dependency.

---

### 8. Nominatim for geocoding

**Choice:** Self-hosted Nominatim instance.

**Motivation:** When a planner enters an order address (Screen 5 — Orders), the system resolves it to lat/lng (required by the `Location` entity in the domain model). Nominatim uses OpenStreetMap data. Self-hosting avoids rate limits, keeps address data internal, and avoids browser-side CORS issues by proxying through the Go API.

**Geocoding flow:**

1. Planner types address in order form.
2. Frontend debounces (300ms), calls `GET /api/geocode?q={address}`.
3. Go API proxies to `GET http://nominatim:8080/search?q={address}&format=json&limit=3&countrycodes=nl`.
4. Returns top results: `[{ lat, lng, display_name }]`.
5. Planner confirms from dropdown; lat/lng is stored with the order.

**Alternatives considered:**

- **Photon**: Lighter-weight geocoder based on OpenStreetMap. Less mature. Could replace Nominatim in a future version.
- **Google Geocoding API**: Accurate but paid and sends address data to Google.
- **Geocoding on save only**: Simpler but gives no real-time feedback to the planner if an address cannot be resolved.

---

### 9. Docker Compose deployment

**Choice:** Single VPS with Docker Compose.

**Motivation:** The application does not require Kubernetes at this scale. All containers run on one host. Caddy handles TLS termination automatically via Let's Encrypt. Docker named volumes persist database and OSRM data across restarts.

**Alternatives considered:**

- **Managed cloud (AWS ECS, GCP Cloud Run)**: More resilient but significantly more complex and expensive at this scale.
- **Bare-metal**: Harder to reproduce, update, and roll back. Docker provides clean service separation.

---

## API Design

The Go server exposes a RESTful JSON API under `/api/`. All endpoints except `/api/auth/*` require a valid JWT cookie.

### Authentication endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/auth/request-code` | Send OTP to email address |
| POST | `/api/auth/verify-code` | Verify OTP, set JWT cookie |
| POST | `/api/auth/logout` | Clear JWT cookie |

### Resource endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/trucks` | List trucks for authenticated user |
| POST | `/api/trucks` | Create truck |
| PUT | `/api/trucks/{id}` | Update truck |
| GET | `/api/orders?date=YYYY-MM-DD` | List orders for a date |
| POST | `/api/orders` | Create order (triggers geocoding) |
| PUT | `/api/orders/{id}` | Update order |
| GET | `/api/plans/{date}` | Get plan for date (auto-generates if unplanned orders exist) |
| POST | `/api/plans/{date}/recalculate` | Re-run VRP solver for date |
| POST | `/api/plans/{date}/finalize` | Set plan status to finalized |
| PUT | `/api/plans/{date}/routes/{routeId}/stops` | Update stop sequence (manual reorder) |
| GET | `/api/geocode?q={address}` | Geocode address via Nominatim proxy |
| GET | `/api/settings` | Get user and company settings |
| PUT | `/api/settings` | Update settings |

### Error responses

All errors return a structured JSON body:

```json
{
  "error": "human-readable error message",
  "code": "machine-readable error code"
}
```

| HTTP status | Code | Situation |
|-------------|------|-----------|
| 400 | `validation_error` | Missing or invalid request fields |
| 401 | `unauthenticated` | No valid JWT cookie |
| 403 | `forbidden` | Resource belongs to another user |
| 404 | `not_found` | Resource does not exist |
| 409 | `conflict` | E.g. plan already finalized |
| 422 | `vrp_infeasible` | No valid plan could be generated |
| 500 | `internal_error` | Unexpected server error |

The frontend maps these codes to user-visible messages matching the error states described in the Functional Design (e.g. "No trucks available", "X orders could not be planned").

---

## Database Schema

The schema directly implements the domain model from the Functional Design. Key implementation decisions:

- All primary keys: UUID, generated by the database (`gen_random_uuid()`).
- Foreign keys: `ON DELETE CASCADE` where child entities have no meaning without the parent (routes without a plan, stops without a route).
- Timestamps: `TIMESTAMPTZ`. Dates: `DATE`. Times: `TIME`.
- All queries scoped to `user_id` to enforce data isolation between planners.

**Indexes:**

```sql
CREATE INDEX idx_orders_user_date ON orders(user_id, date);
CREATE INDEX idx_plans_user_date  ON plans(user_id, date);
CREATE INDEX idx_routes_plan_id   ON routes(plan_id);
CREATE INDEX idx_stops_route_id   ON stops(route_id);
CREATE INDEX idx_locations_user   ON locations(user_id);
CREATE INDEX idx_otp_user_id      ON otp_codes(user_id);
```

**OTP codes table (authentication infrastructure, not in domain model):**

```sql
CREATE TABLE otp_codes (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    code_hash   TEXT        NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL,
    used        BOOLEAN     NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Expired and used codes are cleaned up by a background goroutine running hourly:

```sql
DELETE FROM otp_codes WHERE expires_at < now() OR used = true;
```

---

## Security

### Authentication

- Passwordless OTP: codes stored as bcrypt hashes, invalidated on first use, expire after 10 minutes.
- JWT: signed with HS256, stored in HttpOnly + Secure + SameSite=Strict cookie.
- Rate limiting: 5 OTP requests per email per 15 minutes (token bucket per email in Go middleware).

### Authorisation

Every database query is scoped to the authenticated `user_id` from the JWT. No cross-user data access is possible:

```sql
-- Fetching orders
SELECT * FROM orders WHERE user_id = $1 AND date = $2

-- Updating a truck — 0 rows affected means 404/403
UPDATE trucks SET ... WHERE id = $1 AND user_id = $2
```

### Transport security

- All browser traffic is HTTPS (TLS 1.2+), terminated by Caddy with automatic Let's Encrypt certificates.
- HSTS header set: `Strict-Transport-Security: max-age=31536000`.
- PostgreSQL, OSRM, and Nominatim are only reachable within the Docker bridge network — their ports are not published to the host.

### Input validation

- All inputs validated server-side (Go struct tags + explicit validation). Frontend validation (Zod) is for UX only.
- SQL injection prevention: all queries use parameterised placeholders (`$1, $2, ...`) via `pgx`. No string-interpolated SQL.
- OSRM: coordinates passed are always read from the database (validated on insert), never from raw request bodies.
- Geocoding: the `/api/geocode` endpoint sanitises the query string before proxying to Nominatim.

### CORS

```
Access-Control-Allow-Origin: https://app.routeflow.nl
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
```

Wildcard origin is not used. The allowed origin is configured via an environment variable.

### Secrets management

JWT signing key, Postmark API key, and database password are passed as environment variables. On the production VPS these are stored in a `.env` file with `chmod 600`, read by Docker Compose. They are never committed to the repository.

---

## Scalability and Reliability

*This section describes how the design would change if Routeflow were required to support hundreds of companies, thousands of trucks, and 99.9%+ uptime — significantly beyond the current target scale.*

### Current design limitations

- A single VPS failure takes the entire system offline.
- CPU-bound VRP calculations run synchronously and could block API threads under concurrent load.
- PostgreSQL cannot be scaled independently from the API.
- OSRM and Nominatim share host resources with the API.

### Changes required for extreme scalability

**1. Stateless horizontal API scaling**

The Go API is already stateless (JWT auth, no server-side session). Multiple instances can run behind a load balancer (AWS ALB or Nginx upstream) without modification. The JWT secret would move to a secrets manager (AWS Secrets Manager or HashiCorp Vault).

**2. Managed PostgreSQL with read replicas and failover**

PostgreSQL would move to a managed service (AWS RDS, Supabase, or Neon) with:
- Automated failover (multi-AZ primary/replica).
- Read replicas for read-heavy queries.
- Point-in-time recovery and daily automated snapshots.

Write queries route to the primary; reads route to replicas.

**3. Asynchronous VRP solving (job queue)**

At scale, synchronous VRP solving would block API threads during peak morning hours. The solution:

1. `POST /api/plans/{date}/recalculate` enqueues a job (Redis Streams or RabbitMQ) and returns `202 Accepted`.
2. A pool of dedicated VRP worker containers processes jobs from the queue.
3. The frontend receives the result via Server-Sent Events (SSE) or polling.

**4. OSRM and Nominatim as replicated services**

Both would run as replicated containers behind an internal load balancer. For very high volumes, a commercial routing API (HERE, TomTom) would be evaluated.

**5. Distance matrix caching**

For companies with stable locations (same customers daily), the OSRM distance matrix could be cached in Redis with a TTL. A cache hit avoids the OSRM call entirely.

**6. CDN for the React SPA**

Static assets would be served from a CDN (Cloudflare or AWS CloudFront) instead of a single Nginx container.

**7. Observability**

- **Metrics**: Prometheus + Grafana for API response times, VRP solver duration, error rates.
- **Logging**: Structured JSON logs (Go `slog`) shipped to Loki or Datadog.
- **Tracing**: OpenTelemetry for distributed tracing across API → OSRM → VRP worker.
- **Alerting**: PagerDuty for on-call notifications when error rates spike.

**Summary:**

| Concern | Current (single VPS) | At scale |
|---------|---------------------|----------|
| API instances | 1 | N (behind load balancer) |
| Database | PostgreSQL in Docker | Managed RDS with replicas |
| VRP execution | Synchronous in API | Async job queue + workers |
| OSRM / Nominatim | Single containers | Replicated containers |
| Distance matrix | Computed per request | Cached in Redis |
| Static assets | Nginx on VPS | CDN |
| Observability | Basic logging | Prometheus, Loki, OTel |

---

## Help Received

The following help was received during the creation of this technical design:

- Claude (Anthropic) was used to assist in drafting and structuring this document, based on the Functional Design for Routeflow.
- The OSRM documentation was consulted for details on the `/table` endpoint format and self-hosting setup.
- The OR-Tools documentation was consulted when evaluating VRP solver alternatives.
- The dnd-kit documentation was used when designing the drag-and-drop stop reordering section.

---

*End of Technical Design – Routeflow*