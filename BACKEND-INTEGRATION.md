# Backend Integration Guide

How the `index.html` wireframe screens map to the real Couro backend
(`couro-backend`) and frontend (`couro-react-app`). Written for whoever picks up
turning these mockup screens into real, wired-up UI.

Every claim below is a direct file/line citation from the actual repos as of
2026-07-29 (`couro-react-app` @ `feature/fe1-fe3-squad-upload-drop-height-weight`,
`couro-backend` @ `feat/security-scanning-baseline`, plus one not-yet-merged
backend branch noted explicitly where relevant). Nothing here is guessed —
if a screen has no real endpoint yet, it says so plainly instead of inventing one.

---

## Auth (do this first)

```
POST auth/login          (form-encoded: username, password)
→ { data: { access_token, refresh_token, token_type, user } }
```
`couro-backend/app/api/routes/auth.py:376-469`, `couro-backend/app/models/auth.py:192-214`

Send every subsequent request with:
```
Authorization: Bearer <access_token>
```
The backend also sets httpOnly cookies, but token resolution checks the
`Authorization` header first (`couro-backend/app/api/deps.py:380-405`) — for a
standalone frontend not sharing cookies with the API origin, use the header,
not cookies.

Refresh: `POST auth/refresh`, body `{ refresh_token }` → new `access_token`.

Don't confuse two similarly-named Redux fields on the real frontend:
`state.authUser.tokenAuth.access_token` (the JWT) vs. `state.authUser.token`
(a separate numeric "remaining upload credits" balance, unrelated to auth).

---

## Screen-by-screen

### 1. Squad upload — mapping table + progress view

**Wireframe screens:** `#v-squadupload` (drop/mapping/tagging/progress/results)

**Real frontend today:** `src/pages/squadUpload/SquadUpload.tsx` — this is
**two states on one page**, not a routed wizard: a builder view (mode/date/
speed/position + file drop + `SquadUploadTable` mapping rows to roster
athletes, lines 231-352) that swaps to `SquadUploadProgress` once a batch is
submitted (lines 219-229). Route: `GET /athletes/squad-upload`
(`src/navigation/Router.tsx:151-158`, behind `PrivateRoute` + `TrainerLayout`).

**⚠️ The core gap — read this before wiring anything up:**

The frontend already has a fully-typed hook, `submitSquadBatch()`
(`src/api/query/UseSession.ts:226-238`), that POSTs to `sessions/batch` with
a real multi-athlete payload:
```ts
{ idempotency_key?, job: { service_name, session_date, notification_token? },
  subjects: [{ athlete_id, speed_bucket?, position_profile?, session_name?,
               videos: { side, front?, back? }, selection? }, ...] }
```
**`POST /sessions/batch` does not exist anywhere in the backend** — not on the
checked-out branch, not on any other branch, not even as a stub. I confirmed
this by grepping every route file and every branch.

What *does* exist, on two different, non-overlapping code paths:

- `POST /sessions/start-batch-job` (`couro-backend/app/api/routes/session.py:1107-1355`)
  — real, working, multipart form. But it's **single-athlete**: one
  `athlete_id` field for the whole call, multiple video clips for that one
  athlete. This is not a squad batch in the sense the wireframe means (many
  different athletes in one submission) — it's "upload several clips for one
  athlete at once." Response gives you `session_ids[]` directly; there's no
  batch-status endpoint paired with it — you poll each session id individually
  via `GET /sessions/?session_id=…` or `GET /sessions/athlete/completed`.

- A **separate, mid-construction backend subsystem** (`BatchJob` model,
  `BatchJobRepository`, `BatchController`) exists only on the unmerged branch
  `origin/feature/wbs-r1-5-batchjob`. It has the read side built —
  `GET /sessions/batch/{batch_id}` (`session.py:652-665` on that branch)
  returns exactly the shape the frontend's `getSquadBatchStatus` hook already
  expects (`{ batch_id, state, total, complete, errored, processing, items[], as_of }`)
  — but **the write side that creates a `BatchJob` with multiple athlete
  subjects does not exist yet**, on that branch or any other. The repository
  has a `create_batch_job()` method; nothing in any route calls it. The nested
  `{job, subjects[]}` request shape exists on `/start-job`
  (`couro-backend/app/models/start_job.py`, on the same branch) but that
  handler only ever reads `subjects[0]` — genuinely single-subject today,
  explicitly commented "the shape already carries multi-athlete for the
  M-track" (i.e., designed for this, not built yet).

**Bottom line:** the real multi-athlete squad-batch-create endpoint the
wireframe's squad-upload screen needs is a **real backend gap**, currently
in progress under what looks like a WBS R1-4/R1-5 ticket series. This is not
a frontend task — there's nothing for the frontend to call yet. Recommend
confirming with whoever owns that branch before scoping any frontend work
here; the frontend's own hook is already written and waiting for the
endpoint to exist.

If a squad batch only ever needs to be "one athlete, several clips" (matches
`/start-batch-job` today), that's real and shippable now — but that's a
different feature than what the wireframe's multi-athlete mapping table
implies.

### 2. Dashboard / landing screen

**Wireframe screen:** `#v-dash`

**Real frontend today:** `src/pages/dashboard/index.tsx` is a static
"Select an athlete to view their data" placeholder (lines 68-89) — no roster
data rendered at all. Route: `GET /dashboard`.

**Real backend:** `GET /athlete/trainer/summary`
(`couro-backend/app/api/routes/athlete.py:598-650`) already exists, is fully
built, and returns exactly what this screen needs in one call:
```json
{
  "data": [{
    "athlete_id", "athlete_name", "team_name", "profile_pic_url",
    "session_count", "latest_score", "score_delta", "session_type",
    "archetype": { "name", "confidence" } | null,
    "top_risk": { "name", "body_region", "severity" } | null,
    "last_session_id", "last_session_date"
  }],
  "meta": { "total_count", "visible_count", "athlete_limit", "limit_reached" }
}
```
(`couro-backend/app/controllers/session.py:3523-3632`)

No frontend hook calls this endpoint anywhere today (confirmed via full-repo
grep). **This is the highest-leverage, lowest-risk item on this list** —
backend is done, it's purely a frontend build: add a hook, replace the
placeholder page with the roster-summary-driven landing screen (the
wireframe's quadrant map / KPI tiles are good references for how to present
this exact data).

Two things to handle client-side, not server bugs: `top_risk` and `archetype`
are legitimately `null` on athletes without enough session history yet, and
one athlete's row can silently degrade to defaults if that athlete's
aggregation fails — the endpoint never 500s the whole roster over one bad row.

### 3. Invite / claim

**Wireframe screens:** Settings → "Invite Athlete" modal, claim-links screen

**Real frontend today:** `InviteAthleteModal.tsx`
(`src/components/ui/inviteAthleteModal/`) calls `inviteAthlete()`
(`src/api/query/UseAthletes.ts:144-154`) → `POST athlete/invite`, body
`{ emails: string[], message?: string }` (1-20 emails). This is real and
working. There is no claim-link component or hook anywhere in the frontend.

**Real backend:** `POST /athlete/invite`
(`couro-backend/app/api/routes/athlete.py:98-150`) is real. **There is no
claim-link/claim-token mechanism in the backend at all** — I grepped the
entire controller and model layer for "token"/"claim" and found nothing.
Claiming works purely by **email match**: the invited person signs up or logs
in with the exact email address the invite was sent to, then calls
`PUT /athlete/requests/{request_id}` with `{status: "accepted"}`
(`couro-backend/app/api/routes/athlete.py:185-214`). Authorization is just
`request.athlete_email == user.email` — no token in the URL, no separate
claim flow.

**⚠️ Second real gap:** a trainer's roster upload creates "unclaimed"
athletes via `POST athlete/` — each one gets a brand-new random
`athlete_id = uuid4()`, keyed to nothing. When that same person later accepts
a trainer's invite (matching by email), the backend's accept-handler
(`_set_user_trainer`, `couro-backend/app/controllers/athlete_request.py:555-589`)
only ever looks up or creates a row **keyed by `user.id`** — it never
searches for and merges the pre-existing UUID-keyed roster row for that
email. So today, a roster-uploaded athlete and that same person's
claimed/linked account are **two separate, unreconciled athlete records**.
This is a real backend gap to flag, not something to quietly work around in
the frontend — the wireframe's "claim links" screen (built as a stub/mock in
this mockup) assumes a reconciliation path that doesn't exist yet.

### 4. New Team / org-as-entity

**Wireframe screens:** `#v-newteam`, org overview tab

**Real backend:** confirmed on the checked-out branch — **no
`organization.py` model, no `organization_repository.py`, nothing under
`app/models/` or `app/repositories/` for Team or Company as real entities.**
`company_id` / `team_id` / `company_name` / `team_name` / `company_email` are
unvalidated flat string fields directly on the `Athlete` record
(`couro-backend/app/models/athlete.py:15-39`) — no foreign key, no backing
table, no create/list/manage endpoints. `trainer_id` (one trainer per
athlete) is the only grouping relationship the backend actually enforces.

This is a genuine backend-first item — there's no frontend work that can
build a real "org view" against data that's just loose strings on an athlete
row. (There was a draft `organization.py` in an unmerged agent worktree
mentioned in earlier notes — worth checking if that's progressed, but as of
this branch it doesn't exist in any mergeable form.)

### 5. Single-athlete upload / review screens

**Wireframe screens:** `#v-upload` (movement picker, video, review)

These map to real, working infrastructure already: single-session upload via
`POST sessions/start-batch-job` (single clip = batch of one) or the older
per-session endpoints, session detail via `GET /sessions/?session_id=…`
(returns `{session, ai_coach}`), athlete trends via
`GET sessions/athlete/trends`. No gaps identified here — this is the
"already works, just needs real UI" category.

---

## Data model reference

**`Athlete`** (`couro-react-app/src/interface/Types.ts:13-32`) — matches the
backend's flat shape described above:
```ts
{ athlete_id, athlete_name, profile_pic_url?, height_feet?, height_inches?,
  weight?, gender?, athlete_birthdate?, trainer_id?, company_id?, team_id?,
  trainer_name?, trainer_email?, company_name?, company_email?, team_name?,
  athlete_type?, athlete_email? }
```
No `Team`, `Company`, or `Organization` type exists anywhere in the frontend
either — confirmed via grep. Don't design the real integration around
first-class Team/Org objects; they don't exist yet on either side.

`accountType`: `"individual" | "trainer"` (`Types.ts:316-319`).

---

## Suggested order

1. **Dashboard** — ship first. Backend done, zero dependencies, highest
   visible impact (it's the first screen most coaches land on).
2. **Single-athlete upload/review polish** — real backend already, just UI.
3. **Invite** (as email-invite, not claim-link) — already real end-to-end;
   the wireframe's claim-links screen should be treated as aspirational
   until the roster-upload/invite reconciliation gap (§3) is actually closed
   backend-side.
4. **Squad upload (multi-athlete)** — blocked on real backend work
   (§1). Worth a conversation with whoever owns `wbs-r1-5-batchjob` before
   scoping frontend effort here.
5. **Org-as-entity** — backend-first, no real data model exists yet; scope
   this as its own project, not a frontend task.
