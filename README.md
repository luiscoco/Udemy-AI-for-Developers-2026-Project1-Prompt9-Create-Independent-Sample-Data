# Sample Data Generation

This README documents the steps followed to create the sample data files in
[`data/`](data/), so students can see the reasoning behind each decision, not
just the final output.

## The prompt

The task was to create three fixture files under a root `data/` folder:

- `assets.json` — six fictional assets (HVAC-01, PUMP-04, CNC-02, UPS-03, LIFT-01, COMP-12)
- `technicians.json` — five fictional technicians
- `work-orders.json` — fifteen work orders spread across all six states and four priorities

With these rules:

- References use the `WO-2026-NNNN` format
- Every `assetId` and `technicianId` must point to a real fixture
- Some non-final work orders are intentionally left unassigned
- `reportedAt <= updatedAt` for every work order
- No real people, real customers, or proprietary asset data

## Step 1 — Find the schema before inventing one

Rather than guessing field names and enum values, the codebase was searched
for an existing contract. [`packages/contract/src/types.gen.ts`](packages/contract/src/types.gen.ts)
turned out to be an auto-generated OpenAPI types file that already defines
the exact shape the API expects:

- `WorkOrderState`: `"reported" | "triaged" | "scheduled" | "in_progress" | "completed" | "cancelled"` (6 states)
- `Priority`: `"low" | "medium" | "high" | "critical"` (4 priorities)
- `Asset`: `id`, `tag`, `name`, `location`
- `Technician`: `id`, `name`, `specialty`
- `WorkOrder`: `id`, `reference`, `assetId`, `title`, `description`, `priority`, `state`, `technicianId`, `reportedAt`, `updatedAt`

Using the generated contract instead of improvising field names means the
fixtures will actually match what the Fastify API and React app expect to
read.

**Lesson:** when a task says "create data" in a typed project, look for the
type/schema definitions first — they're the real spec, and skipping this
step is the most common way to produce data that "looks right" but doesn't
load.

## Step 2 — Design the assets and technicians

Six assets were created, one per required tag, each with a short descriptive
`name` and a plausible `location` (building/room/bay), all fictional.

Five technicians were created with fictional names and a `specialty` field
(Electrical, Mechanical, HVAC, Hydraulics, Controls & Automation) so that
later technician assignments could look domain-appropriate (e.g. an HVAC
specialist assigned to an air handler unit).

IDs were written as fixed, readable UUID-shaped strings (e.g.
`11111111-1111-4111-8111-111111111101`) rather than random UUIDs, so that
cross-references between files stay easy to read and verify by eye during
review.

## Step 3 — Design the fifteen work orders

To satisfy "spread across all six states and four priorities" without just
scattering values randomly, the 15 work orders were planned as a small table
first:

| # | State | Priority | Assigned? |
|---|-------|----------|-----------|
| 1–3 | `reported` | critical / medium / low | unassigned |
| 4–6 | `triaged` | high / medium / low | mixed |
| 7–9 | `scheduled` | critical / medium / low | mixed |
| 10–12 | `in_progress` | high / critical / medium | assigned |
| 13–14 | `completed` | low / high | assigned |
| 15 | `cancelled` | medium | unassigned |

This guarantees every state and every priority appears at least once (in
fact multiple times), while keeping the distribution readable instead of
arbitrary.

**Unassigned technicians:** work orders in early, non-final states
(`reported`, `triaged`, `scheduled`) were deliberately left with
`technicianId: null` in several cases — reflecting a realistic workflow
where new or triaged tickets haven't been staffed yet. One `cancelled`
order was also left unassigned, representing a duplicate ticket that was
closed before anyone was assigned to it. `in_progress` and `completed`
orders are always assigned, since a technician must exist for work to have
started or finished.

**Dates:** `reportedAt` and `updatedAt` were chosen so that
`reportedAt <= updatedAt` always holds, and so the timestamps tell a
plausible story — e.g. `completed` orders have older `reportedAt` dates and
an `updatedAt` a few days later (when the work wrapped up), while freshly
`reported` orders have matching or near-matching timestamps.

**References:** each work order was numbered sequentially as
`WO-2026-1001` through `WO-2026-1015`.

## Step 4 — Validate referential integrity

Rather than eyeballing the JSON, a small Node.js script was run against the
three generated files to check:

- Every `assetId` in `work-orders.json` exists in `assets.json`
- Every non-null `technicianId` exists in `technicians.json`
- `reportedAt <= updatedAt` for every work order
- `reference` matches the `WO-2026-NNNN` pattern
- All 6 states and all 4 priorities are represented at least once

The script confirmed all checks passed before the files were considered
done. This is the same principle as writing a test after implementing a
feature: generating data by hand is easy to get subtly wrong (a typo'd ID,
a swapped date), so a quick automated check is cheap insurance.

## Files produced

- [`data/assets.json`](data/assets.json)
- [`data/technicians.json`](data/technicians.json)
- [`data/work-orders.json`](data/work-orders.json)

## Running the project (current state)

At this point in the course, `apps/backend` and `apps/frontend` are still
empty scaffold packages — each has a `package.json` that depends on
`@equipment-hub/contract`, but no source code and no Fastify/Vite wiring
yet. The root `dev` script is a placeholder
(`echo "no dev server configured yet"`), so there is no "start the app"
command to run yet — that gets added in a later prompt.

What you *can* run today, from the repo root in a Windows terminal
(PowerShell or Command Prompt):

```powershell
# Install dependencies for all workspaces (assets/backend/frontend/contract)
npm install

# Lint the whole repo
npm run lint

# Verify the OpenAPI contract used to generate packages/contract/src/types.gen.ts
npm run verify:contract
```

Once the backend and frontend apps are scaffolded with real dev servers in
a future prompt, this section will be updated with the actual `npm run dev`
(or equivalent) command to launch the app.

## Takeaways for students

1. **Read the types before writing the data.** A generated contract file
   (`*.gen.ts`, OpenAPI schema, Zod schema, etc.) is the source of truth for
   field names and enums — don't infer them from the prompt alone.
2. **Plan distributions with a table**, especially when a prompt says
   "spread across all X and Y" — it's easy to accidentally miss a state or
   priority when improvising row by row.
3. **Encode business rules as deliberate choices**, not randomness (e.g.
   "in-progress work always has a technician" is a modeling decision, not
   an accident).
4. **Validate generated fixtures programmatically** — a short script
   catches referential and ordering mistakes that a visual review can miss.
