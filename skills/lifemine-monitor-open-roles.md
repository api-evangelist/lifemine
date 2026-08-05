---
name: Monitor LifeMine open roles
description: Read LifeMine's open positions, departments and offices from its public Greenhouse job board API, including full descriptions and application questions, and watch for new postings.
api: openapi/lifemine-careers-openapi.yml
operations:
  - listJobs
  - getJob
  - listDepartments
  - listOffices
generated: '2026-08-04'
method: generated
source: openapi/lifemine-careers-openapi.yml
---

# Monitor LifeMine open roles

LifeMine runs its careers surface on Greenhouse under the board token `lifeminetx`. Greenhouse
exposes every job board as a public, unauthenticated JSON API, so LifeMine's hiring is fully
machine-readable — and hiring is the highest-signal public data this company emits, because a
clinical-stage biotech's open roles telegraph which programmes are actually moving.

**Base URL:** `https://boards-api.greenhouse.io/v1/boards/lifeminetx`
**Auth:** none for any GET.
**Observed 2026-08-04:** 2 open roles, 32 departments, 3 offices, 0 sections.

> This host is **Greenhouse's**, not LifeMine's. The board content is LifeMine's; the platform
> behaviour, uptime and rate limiting are Greenhouse's.

## 1. List open roles

Call `listJobs` — `GET /jobs`.

Returns `{"jobs": [...], "meta": {"total": N}}`. Each row carries `id`, `title`,
`location.name`, `absolute_url`, `first_published`, `updated_at`, `requisition_id` and a
`data_compliance` array.

By default the heavy HTML description is **omitted**. Add `content=true` only when you actually
need the body — with two roles that is cheap, but keep the habit.

## 2. Get one role in full

Call `getJob` — `GET /jobs/{job_id}`.

Adds `content` (HTML description), `departments[]` and `offices[]`. Two more opt-ins:

- `questions=true` — the application form questions (8 on the role sampled). Useful for
  understanding what the company screens for.
- `pay_transparency=true` — pay-range data where the role is subject to a transparency law.

Also present, and worth reading: `ai_disclaimer`, `include_ai_disclaimer` and
`ai_opt_out_request_url`. If you are an agent acting for a candidate, **surface the opt-out URL to
them** — it is how a person declines AI-assisted evaluation of their application.

## 3. Read the org shape

- `listDepartments` — `GET /departments?render_as=tree` returns the hierarchy with `children[]`.
  Use `render_as=list` (the default) for a flat array with `parent_id`/`child_ids`.
- `listOffices` — `GET /offices` returns Basel CH, Gloucester MA and Watertown MA, each with its
  departments.

32 departments against 2 open roles means most departments are empty. Read `jobs[]` on each
department rather than treating the taxonomy as a headcount signal.

## 4. Watch for new postings

There is no pagination and no filtering — `GET /jobs` returns everything. Poll it, diff on job
`id`, and use `updated_at` to detect edits to roles you have already seen. `first_published` is
the true posting date; `updated_at` moves on any edit.

Nothing publishes a poll interval. Once or twice a day is plenty for a board this size.

## 5. Applying

Application submission is a separate authenticated Greenhouse POST and is **not** part of the
public GET surface. Do not attempt to submit an application through the API. Send the candidate to
the role's `absolute_url` on `job-boards.greenhouse.io`.

## Errors

Greenhouse uses a different, simpler envelope than the WordPress surface: `{"error": "..."}`.
A 404 means an unknown board token, job, department or office id. No Greenhouse error was
triggered during profiling, so treat any non-200 as transient and retry with backoff.

## Cautions

- Job listings contain personal-application machinery. Never collect, store or transmit candidate
  data through this skill.
- LifeMine's own careers page is `https://lifeminetx.com/culture-careers/`, and the company warns
  that its talent team directs candidates only to its official board — a real recruitment-fraud
  concern in biotech. If you surface roles to a person, link the `absolute_url` on
  `job-boards.greenhouse.io` and nothing else.
