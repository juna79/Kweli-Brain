[Uploading KWELI_BRAIN_V2_README.md…]()
# Kweli Brain V2 — README

Built: 10 July 2026, from a full read of V1's `index.html` (4,231 lines) and
the 10 July 2026 Brain Snapshot. This is a clean rebuild, not a patch — see
"What changed and why" below.

## Files in this delivery

| File | Purpose |
| --- | --- |
| `kweli_brain_v2_schema.sql` | Full normalised schema — 20 tables, RLS, triggers, indexes |
| `kweli_brain_v1_to_v2_migration.sql` | Restartable migration from V1's actual tables |
| `kweli_brain_v2.html` | The V2 application |
| `KWELI_BRAIN_V2_README.md` | This file |

There is **no separate migration-helper HTML file**. V1's "hidden JSON in a
text column" trick (followups packed into `opportunities.notes`, person
metadata into `people.notes`, board replies into `team_messages.message`)
turned out to be fully parseable with plain Postgres string functions and
`jsonb`, so the parsing lives in `legacy_unpack()` inside the migration SQL
itself, running server-side in one transaction. That's more reliable than a
browser step (no partial-completion risk if a tab closes) and is still fully
restartable and dry-run-able — it just doesn't need its own HTML file. If you
disagree and want a browser-based review UI for the `migration_log` output,
that's a small follow-up; the report itself already exists (see below).

## Architecture summary

**One record, multiple views.** Every organisation, person, opportunity,
meeting, task, asset, lesson, objection and decision is a real row with its
own table and foreign keys — never JSON hidden inside a text column, never a
second copy of the same fact. A universal `entity_links` table carries
relationships that don't fit a foreign key (asset → opportunity, lesson →
meeting, etc.), with an `is_inferred` flag so suggested links (e.g. "this
asset title mentions Kentaste") are never silently treated as confirmed.

**`activities` is the master timeline.** Completing a follow-up, saving
meeting notes, uploading an asset — each writes exactly one row to
`activities`, tagged with every entity it's relevant to
(`organisation_id`, `person_id`, `opportunity_id`, `task_id`, `meeting_id`,
`asset_id`...). The organisation timeline, person timeline, opportunity
timeline, global search and the AI Workspace all read this same table. There
is nothing to keep in sync because there's only one copy.

**One calendar engine.** `calendarItemsFor(year, month)` in the app pulls
from `tasks`, `meetings` and `marketing_items` in a single pass. The
Follow-ups page is a management view of the same `tasks` table, not a second
calendar.

## Setup order

1. **Back up V1.** In Supabase: Database → Backups, or `pg_dump` the
   project. V1's tables are never touched destructively by anything in this
   delivery, but back up anyway before running SQL against a production
   project.
2. **Run `kweli_brain_v2_schema.sql`** in the Supabase SQL editor, in full,
   even if V1's tables (`opportunities`, `people`, `lessons`, `assets`,
   `team_messages`) are still sitting in `public`. Step 0 of the file
   detects and moves them into a `legacy` Postgres schema automatically
   (a rename, not a copy — nothing is duplicated or dropped) before creating
   any V2 table, so there's no naming collision. It's idempotent — safe to
   re-run if something else fails partway through.
   - **If you already ran an older copy of this file and hit `column
     "primary_organisation_id" does not exist"` or similar** — that was this
     exact sequencing issue in an earlier version. Just re-run the current
     file in full; Step 0 will now move the leftover V1 tables out of the
     way and the rest of the script will complete cleanly.
3. **Create the `assets` storage bucket** if you're using a fresh project
   (skip if reusing V1's project — it already exists). Keep it **private**.
   Apply the four storage policies commented at the bottom of the schema
   file.
4. **Run `kweli_brain_v1_to_v2_migration.sql`**, wrapped in an explicit
   transaction so you can dry-run it first:
   ```sql
   begin;
   -- paste/run the migration file
   select step, status, count(*) from migration_log group by step, status order by step;
   -- if it looks right:
   commit;
   -- if not:
   rollback;
   ```
   This file also opens with a safety check that fails fast with a clear
   message if `kweli_brain_v2_schema.sql` hasn't fully completed — so if you
   see that error, go back to step 2, not this file.
5. **Review the migration report**:
   ```sql
   select * from migration_log where status in ('needs_review','duplicate','error') order by step;
   ```
   This flags: possible duplicate organisations (name-similarity match, e.g.
   Kentaste/Kentatse), people that migrated without an organisation link,
   V1's free-text `objections` field migrated as one record per opportunity
   (split it if it actually covers more than one), and Team Board `readBy`
   entries that couldn't be safely mapped to a V2 user ID (see "Known
   limitations").
6. **Point `kweli_brain_v2.html` at the same Supabase project** (the URL/key
   at the top of the file already match V1's — update only if you migrated
   into a different project) and open it in a browser.
7. **Create `user_profiles` rows** for Juna/Arjun, Paris and Mason — they're
   created automatically on first sign-in (via Supabase Auth), so just have
   each person sign in once.
8. **Validate against the 10 July snapshot** — spot-check that Verst Carbon,
   Kentatse, Dalberg Research and Moringa School all appear as organisations
   with their opportunities and next actions intact, and that the Verst
   discovery-meeting notes appear in Verst's timeline.
9. **Run the test checklist below.**
10. Only after validation, retire V1 (or leave `legacy.*` in place
    indefinitely — it costs nothing to keep).

## Migration report — how to read it

`migration_log` has one row per record processed, per step, with a `status`
of `ok`, `skipped`, `duplicate`, `needs_review`, or `error`. Nothing is ever
deleted from V1, and re-running the migration script is safe: every insert
is gated on "does a `migration_log` row already exist for this source
record" so nothing is duplicated on a second run.

## RLS

One internal policy per table: any authenticated user can read and write
(see the `do $$ ... $$` block at the end of the schema file). This matches
the spec — a small internal team, not external users. If Kweli ever gives
an external party (an auditor, a design-partner contact) direct database
access, tighten this to per-owner or per-role policies before that happens;
right now nothing enforces that only Juna/Paris/Mason can write.

## Rollback plan

- V1's tables live untouched in the `legacy` schema after migration — point
  V1's `index.html` at `legacy.<table>` (a one-line change to each
  `sb.from('table')` call) to bring V1 back online if V2 has a problem.
- V2's own tables can be dropped and re-migrated from `legacy.*` at any
  time — the migration script is restartable and non-destructive to its
  source.
- Nothing in this delivery drops a V1 table. The only "point of no return"
  is if you manually delete the `legacy` schema — don't, until you're
  confident V2 is fully validated.

## Test checklist (from the build brief's 10 scenarios)

| # | Scenario | Status |
| - | --- | --- |
| 1 | Company timeline (Verst follow-up + meeting notes → one activity, visible everywhere, no duplicates) | **Implemented** — `saveMeetingNotes()` / `completeTask()` write exactly one `activities` row |
| 2 | Waiting state (email Rehan → task becomes Waiting for Response, stays on calendar) | **Implemented** — `completeTask()` outcome mapping |
| 3 | People deduplication (Kyle Denning saved twice → one record) | **Partially implemented** — DB-level unique index on email/LinkedIn blocks exact dupes; the create-person form also warns on a probable match before saving. Name-only duplicates (no email/LinkedIn on either record) are **not** blocked — flagged in `migration_log`/needs manual review instead, per the spec's own instruction not to silently merge on name alone |
| 4 | Asset inference (upload "Kweli x Kentaste" → suggested link, not automatic) | **Implemented** — `uploadAsset()` creates `is_inferred:true` links; `confirmAssetLink()` requires a click to accept |
| 5 | Lessons (created from meeting notes, linked to meeting/org, visible in both places) | **Implemented** — `saveMeetingNotes()` checkbox path |
| 6 | Calendar (prev/next month, one engine for tasks/meetings/marketing) | **Implemented** |
| 7 | Team Board colours (yellow/red/green, done overrides priority) | **Implemented** — `tbColor()` |
| 8 | Team Board realtime (Paris sees an assigned note without refresh; independent per-user read state) | **Implemented** — Supabase Realtime subscription on `team_messages`/`team_message_replies`; read state via `team_message_reads` keyed by `user_id` |
| 9 | AI retrieval (focused sources only, no unrelated companies) | **Implemented at the retrieval layer** — `resolveEntities()` + `runAsk()` return only linked records. No LLM is connected (`callLLM()` returns `null` by design — see below), so there is no generated prose answer yet, only the retrieved context block and a "Copy to Claude" button |
| 10 | Snapshot export (accurate, no duplicates, no hidden JSON) | **Implemented** — `buildSnapshotMarkdown()` reads live from `state`, which mirrors the V2 tables directly |

## Known limitations

Being direct about what's real versus scaffolded, per how this project runs:

- **The AI Workspace does not call a model.** `callLLM()` is a deliberate
  stub returning `null`. Wiring it up requires a server-side function (see
  below) — doing that without one would mean either faking reasoning or
  putting an API key in browser-visible code, both of which the build brief
  explicitly rules out.
- **Team Board note dragging (`board_x`/`board_y`) has columns but no drag
  UI yet.** The migration preserves V1's saved positions where they existed;
  the V2 app currently opens the board as a grid, not a freely-positioned
  cork-board. Positions are not lost, just not yet interactively editable.
- **PDF text extraction is schema-ready, not implemented.** `assets` has
  `extracted_text` and `text_extraction_status` columns and the migration
  carries over anything V1 already extracted, but there's no extraction job
  in V2 yet — `text_extraction_status` stays `not_started` for anything new.
  Search does not search inside PDF contents until that job exists.
- **`investor_profiles` has no create/edit form in the app yet** — the page
  reads the table but there's no "New investor" button. The table and
  migration are ready; add the form when investor tracking becomes active
  again (per your notes, the current live pipeline is climate/ESG-heavy, not
  investor-heavy, so this was deprioritised — flag if that's wrong).
- **Detail-page sub-tabs are read-heavy.** Every tab on the organisation page
  works and pulls live data, but a few (Assets tab, Decisions tab) don't yet
  have their own "add" button *from that specific tab* — add via the Assets
  or Decisions page instead for now.
- **Name-only duplicate people are not blocked by the database**, only
  flagged during migration (see test 3 above) — this is a deliberate
  trade-off, not an oversight: two different people can share a name, so a
  hard DB constraint on name alone would be worse than the current
  soft-warning approach.
- **Meeting attendees UI is minimal.** The `meeting_attendees` table and
  join exist; the meeting-notes modal doesn't yet have an attendee picker —
  attendees currently have to be added via a direct insert or a small
  follow-up.

## How to connect an LLM later without exposing an API key

Never call the Anthropic or OpenAI API directly from `kweli_brain_v2.html`
with an embedded key — anyone who views source gets it. Instead:

1. Create a Supabase Edge Function (`supabase functions new ask-kweli`).
2. Store the API key as a Supabase secret (`supabase secrets set
   ANTHROPIC_API_KEY=...`), never in the repo or the HTML.
3. The Edge Function receives `{question, context}` from the browser (the
   same `context` string `runAsk()` already builds), calls the model
   server-side, and returns the answer.
4. Replace the stub in `kweli_brain_v2.html`:
   ```js
   async function callLLM(question, context){
     const {data,error} = await sb.functions.invoke('ask-kweli', {body:{question, context}});
     if(error) return null;
     return data.answer;
   }
   ```
5. In `runAsk()`, call `callLLM()` and render its result above the raw
   context block if it returns non-null — the "Copy to Claude" fallback
   stays either way, since it costs nothing to keep.

## What changed and why (V1 → V2, in brief)

- **Hidden JSON in notes columns → real tables/columns.** V1's own code
  comments describe this as a deliberate workaround for not being able to
  change the Supabase schema at the time ("packed as a hidden JSON block...
  stripped back out on read"). V2 removes the need for the workaround by
  having the real columns from the start.
- **`opportunities.company` (free text) → `organisations` table with a
  proper foreign key.** This is what makes "one organisation, multiple
  opportunities" possible (Kentaste's Farmer App opportunity and a future
  second Kentaste opportunity are now two rows sharing one organisation, not
  two unrelated opportunity records that happen to have the same company
  string).
- **`timeline_events` (opportunity-only) → `activities` (org/person/
  opportunity/task/meeting/asset/lesson/decision/team-message, all
  optional foreign keys on one row).** This is the single biggest change —
  it's what makes a person's timeline, an opportunity's timeline and a
  meeting's activity all the same underlying data instead of three separate
  copies.
- **Follow-ups (packed array inside `opportunities.notes`) → `tasks`
  table** with its own status lifecycle, so a follow-up's history survives
  independently of whatever else changes on the opportunity.
- **Team Board `message` column (packed replies/reads/position) →
  `team_messages` + `team_message_replies` + `team_message_reads`**, so
  read-state is genuinely per-user (a `(team_message_id, user_id)` primary
  key) instead of an array inside one shared text field.

## Rollback / support

If something in the migration looks wrong before you `commit`, just
`rollback` — nothing is written. If you've already committed and find an
issue, the source data in `legacy.*` is untouched, so the fix is: truncate
the specific V2 table(s) affected, delete the corresponding
`migration_log` rows for that step, and re-run that step's block from the
migration file.
