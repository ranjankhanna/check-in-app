# Temple Check-In App — Master Design Document

**Status:** Final implementation baseline. Reviewed and finalized August 12, 2026.

---

## 1. Purpose

An iOS/Android app plus a self-service iPad kiosk, used to check in temple members and their families to events. Supports both members who pre-registered for an event and walk-ins who haven't. Designed to handle families/groups checking in together with one interaction, not just individuals.

---

## 2. Check-in methods

A person can identify themselves three ways:
1. **QR code** on their own mobile device (personal app)
2. **Phone number or email lookup**
3. **Physical card** with a QR/barcode — planned for a future phase, not yet built

---

## 3. Two contexts, one core logic

The same identify → household → event → confirm logic runs in two different shells:

| | Personal app | iPad kiosk |
|---|---|---|
| Usage | Self-service, on the user's own phone | Self-service, unattended, volunteer available nearby |
| Default state | Home screen | Idle / attract screen (logo, today's events, "tap to begin") |
| Identify | Camera opens automatically for personal QR; phone/email fallback | Equal weight given to QR scan and phone/email search |
| Household screen | Shown after match | Same, but with large touch targets |
| Reset behavior | N/A — single user's own device | **Auto-resets to idle** after ~4s post-confirmation, or after any inactivity mid-flow — this is a privacy requirement, not just UX, since the next person in line must never see a prior family's data |
| Device lockdown | N/A | Needs kiosk lock mode (iOS Guided Access or MDM) so it can't be swiped out of the app |

---

## 4. Core user flow

1. **Identify** — QR scan, barcode scan, or phone/email lookup.
2. **Match found?**
   - **No** → simple message: *"We couldn't find a match. Please check with the front desk for help."* → returns to idle. No in-app volunteer-notification system — kept simple for v1.
   - **Yes** → load the person's household (all users sharing the same `household_id`).
3. **Household checklist** — "Who's here today?" — all household members shown, pre-checked (most common case: whole family present). User unchecks anyone not present. Any member's QR/lookup surfaces the *whole* household, not just themselves — lower friction than requiring a specific person to scan.
4. **Event resolution** (see Section 6 for full detail):
   - Auto-fills from existing registrations where possible, shown as an **editable pre-checked confirmation** (never silently auto-submitted).
   - Falls back to a manual pick-list for unregistered walk-ins.
   - Events requiring registration with no registration on file are shown **locked**, not selectable — routes to front desk.
5. **Confirm** — one `check_ins` row written per person, per event, all sharing one `checkin_session_id` to group the household's visit together.

---

## 5. Data model (Supabase / Postgres)

Full schema lives inline below. Summary:

### households — no longer a table
There is **no `households` table.** `household_id` lives as a plain UUID column directly on `users`, with no foreign key and no backing table (Option B, chosen deliberately over keeping a minimal table for referential-integrity enforcement — for a project of this size and single-maintainer scope, the enforcement wasn't judged worth the extra table and registration step).

- Grouping is just a shared UUID value: every member of the same household has the same `household_id`. There is no database-level enforcement that household IDs were issued by the registration process or assigned consistently. Household grouping integrity is therefore the application/registration process's responsibility.
- No `household_name` — removed earlier. It served no functional purpose beyond a display label; the checklist screen shows member names directly with no group heading.
- No `primary_contact_user_id` / "head of household" concept — removed. Not needed by any current feature.
- No `primary_phone` / `primary_email` at the group level — removed earlier for the same reason (ambiguous whose number, out of several members). Lookups match against a specific `users.phone_normalized` / `users.email_normalized`, then resolve to that person's `household_id`.

### users
Individual member records — the atomic unit everywhere (registration, check-in, reporting).

- `household_id` — plain UUID, shared across members of the same household, no FK (see above).
- `qr_code` — opaque unique code, this is what's embedded in a person's QR/barcode — **not** their raw `id`. Keeps physical card issuance later a pure "generate code for user_id" operation with no schema change.
- `phone_normalized` / `email_normalized` — canonical lookup values maintained by the backend. Raw `phone` / `email` remain display values. Email normalization is lowercase + trimmed; phone normalization stores digits in a canonical international form defined by the registration/import process. Lookups use only the normalized columns, never raw free-form values. Phone/email lookup may return multiple users: if all matches share the same `household_id`, the backend treats the result as one unambiguous household match; if matches span multiple household IDs, it returns an `ambiguous_match` result, exposes no member details, and directs the user to the front desk.
- `is_dependent` — true for kids/members without independent contact info.
- `parent_1_id`, `parent_2_id`, `spouse_id` — self-referencing relationship columns (see Section 7). Application/backend validation prevents self-reference and duplicate parent assignments; reciprocal spouse maintenance is handled by the owning registration/admin process rather than database triggers.

### events
- `requires_registration` — gates whether walk-ins are allowed (see Section 6).
- `checkin_opens_at` / `checkin_closes_at` — explicit eligibility window for check-in. If omitted by the upstream event-management process, they default conceptually to `start_time` / `end_time`; production ingestion should populate them explicitly for events that allow early arrival or late entry.
- No payment-related columns — payment is explicitly out of scope (see Section 8).

### event_registrations
**Externally populated — this app only reads from this table, never writes to it.** Represents registration intent from a process/team outside this system's ownership. Kept separate from `check_ins` (attendance) so sign-up-to-attendance conversion can be measured per event.

### check_ins
One row per person, per event, per attendance record. `checkin_session_id` groups a household's simultaneous check-in together even across multiple events. `method` records how identity was resolved (`qr` / `barcode` / `phone_lookup` / `email_lookup`).

**Duplicate / re-entry rule:** v1 treats check-in as idempotent per person + event. A second scan for someone already checked into that event does **not** create a new attendance row; the API returns an `already_checked_in` result and the UI confirms that the person is already checked in. Re-entry counting is intentionally out of scope for v1. This is enforced with a unique constraint on (`user_id`, `event_id`).

### Full schema (Supabase / Postgres, ready to run)

```sql
-- =========================================================
-- Temple Check-In App — Database Schema (Supabase / Postgres)
-- =========================================================
-- Design decisions reflected here:
--  - No households table. household_id is a plain shared UUID on
--    users, with no foreign key or backing table (Option B — chosen
--    over a minimal enforcement-only table for this project's scope).
--  - No household-level phone/email (ambiguous whose number it was).
--    Lookups match against a specific user's own normalized phone/email.
--  - Relationships (parent/spouse) stored as simple self-referencing
--    columns on users, supporting unlimited generations via chained
--    lookups or a recursive query — no separate relationships table.
--  - Row Level Security should be enabled with access only via a
--    service role / backend layer, not directly from client apps,
--    since this table holds personal contact information.
-- =========================================================

create extension if not exists pgcrypto;

-- ---------------------------------------------------------
-- Users
-- ---------------------------------------------------------
create table users (
  id uuid primary key default gen_random_uuid(),

  -- Household grouping: plain shared UUID, no FK, no backing table.
  -- Every member of the same household carries the same value here.
  -- Integrity (a valid, consistent grouping) is the application's
  -- responsibility, not the database's.
  household_id uuid not null,

  first_name text not null,
  last_name text not null,
  phone text,
  phone_normalized text,
  email text,
  email_normalized text,
  is_dependent boolean not null default false,
  date_of_birth date,

  -- Relationships (self-referencing; supports multi-generation via chaining)
  parent_1_id uuid references users(id),
  parent_2_id uuid references users(id),
  spouse_id uuid references users(id),

  qr_code text unique,           -- opaque code embedded in the user's QR/barcode
  registered_at timestamptz not null default now()
);

create index idx_users_phone_normalized on users(phone_normalized);
create index idx_users_email_normalized on users(email_normalized);
create index idx_users_household on users(household_id);
create index idx_users_parent_1 on users(parent_1_id);
create index idx_users_parent_2 on users(parent_2_id);
create index idx_users_spouse on users(spouse_id);

-- ---------------------------------------------------------
-- Events
-- ---------------------------------------------------------
create table events (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  description text,
  start_time timestamptz not null,
  end_time timestamptz not null,
  requires_registration boolean not null default false,
  checkin_opens_at timestamptz,
  checkin_closes_at timestamptz,
  created_at timestamptz not null default now(),
  check (checkin_opens_at is null or checkin_closes_at is null or checkin_opens_at <= checkin_closes_at)
);

create index idx_events_time_window on events(start_time, end_time);

-- ---------------------------------------------------------
-- Event registrations (intent — signed up ahead of time)
-- ---------------------------------------------------------
-- NOTE: This table is populated by an external process/team, not by
-- this app. The check-in system only ever READS from it — it never
-- writes registration rows. Payment is entirely out of scope for this
-- app and is not represented anywhere in this schema; this app only
-- asks "does a registration row already exist for this person + event?"
create table event_registrations (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references users(id) on delete cascade,
  event_id uuid not null references events(id) on delete cascade,
  registered_at timestamptz not null default now(),
  unique (user_id, event_id)
);

create index idx_registrations_user on event_registrations(user_id);
create index idx_registrations_event on event_registrations(event_id);

-- ---------------------------------------------------------
-- Check-ins (actual attendance — one row per person, per event)
-- ---------------------------------------------------------
create table check_ins (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references users(id) on delete restrict,
  event_id uuid not null references events(id) on delete restrict,
  checkin_session_id uuid not null,  -- groups a household's check-in together
  method text not null check (method in ('qr', 'barcode', 'phone_lookup', 'email_lookup')),
  checked_in_at timestamptz not null default now(),
  checked_in_by text,                 -- device_id or staff identifier, nullable
  client_request_id uuid not null      -- shared by all rows in one submitted check-in session
);

alter table check_ins
  add constraint uq_checkins_user_event unique (user_id, event_id);

create index idx_checkins_user_event on check_ins(user_id, event_id);
create index idx_checkins_session on check_ins(checkin_session_id);
create index idx_checkins_client_request on check_ins(client_request_id);

-- Relationship sanity checks enforced in the database where straightforward
alter table users add constraint chk_users_not_own_parent_1 check (parent_1_id is null or parent_1_id <> id);
alter table users add constraint chk_users_not_own_parent_2 check (parent_2_id is null or parent_2_id <> id);
alter table users add constraint chk_users_not_own_spouse check (spouse_id is null or spouse_id <> id);
alter table users add constraint chk_users_distinct_parents check (parent_1_id is null or parent_2_id is null or parent_1_id <> parent_2_id);

-- =========================================================
-- Example queries this schema supports
-- =========================================================

-- All members of a household
-- select * from users where household_id = :household_id;

-- A child's parents
-- select * from users where id in (
--   select parent_1_id from users where id = :child_id
--   union
--   select parent_2_id from users where id = :child_id
-- );

-- A child's grandparents (two-hop)
-- select g.* from users child
-- join users parent on parent.id = child.parent_1_id or parent.id = child.parent_2_id
-- join users g on g.id = parent.parent_1_id or g.id = parent.parent_2_id
-- where child.id = :child_id;

-- Full ancestor chain, any depth (recursive)
-- with recursive ancestors as (
--   select id, parent_1_id, parent_2_id, 1 as depth
--   from users where id = :starting_user_id
--   union all
--   select u.id, u.parent_1_id, u.parent_2_id, a.depth + 1
--   from users u
--   join ancestors a on u.id = a.parent_1_id or u.id = a.parent_2_id
-- )
-- select * from ancestors where depth > 1;
```

---

## 6. Event selection — detailed logic

### Check-in eligibility window
An event is offered only while the current time falls inside its effective check-in window. Resolve each boundary independently: `effective_checkin_opens_at = coalesce(checkin_opens_at, start_time)` and `effective_checkin_closes_at = coalesce(checkin_closes_at, end_time)`. This makes early-arrival or late-entry policy explicit, supports a single overridden boundary without ambiguity, and avoids relying on an ambiguous definition of "today." Events outside the effective window are not selectable.

### Single vs. multiple concurrent events
Households can check into **more than one event running at once** (e.g., parents at "Sunday Satsang," kids at "Kids Program" simultaneously).

- **Default UI**: a simple multi-select list, same for the whole household — covers the common case where everyone's attending the same thing(s).
- **"Not everyone going to the same thing?"** — a lightweight toggle beneath the list, only used in the minority case where the household is splitting up. Reveals a **one-person-at-a-time screen** (not a matrix/grid) — each person sees their own simple checklist, swipe/next between people.

### Auto-fill from registrations
- If a person has a matching registration for an **eligible event**, it's shown **pre-checked but editable** on a confirmation screen — never silently auto-submitted. People should always see and be able to change what they're about to be checked into.
- Zero registrations → blank, selectable list.
- Multiple registrations → multiple pre-checked entries, still editable.

### Registration-required events, no registration on file
**Blocked, not allowed as a walk-in.** Shown in the list as locked/disabled with a reason:
```
🔒 Community Lunch — requires registration, check front desk
```
This is per-person — one household member may be registered while another isn't; each sees their correct state independently. If tapped, shows: *"[Event] requires registration. Please check with the front desk."* → returns to the event list (not idle — they may still check into other, unrestricted events).

### Gating logic (pseudocode)
```
For a given person + event:
  if current_time is outside the event check-in window
    → not offered / not selectable
  else if event.requires_registration and no row in event_registrations
    → locked, "requires registration, check front desk"
  else
    → allowed
```

---

## 7. Family / relationship model

Two distinct concepts, deliberately kept separate:

| | `household_id` (shared grouping value) | `parent_1_id` / `parent_2_id` / `spouse_id` |
|---|---|---|
| Answers | "Who checks in together?" | "Who is related to whom?" |
| Nature | Operational, practical | Factual, persists across life changes |
| Can include | Unrelated people (roommates, caretakers, family friends) | Only actual family relationships |
| Changes when | Living/attendance situation changes | Rarely — biological/legal facts |
| Stored as | Plain shared UUID on `users`, no backing table | Self-referencing FKs on `users` |

**Why not a single unified concept:** an adult child with their own family is still biologically Raj's child (`parent_1_id` points to Raj) but is a completely separate household with their own kids — deriving "household" from relationship chains alone has no natural stopping point.

**Why not a full relationships table (rejected for now):** a dedicated `household_relationships` table (storing typed pairwise links like `spouse`, `parent_of`, `sibling_of`) was designed but set aside as unnecessary complexity — nothing in the current app logic needs to query exact relationship types. The simple self-referencing columns are sufficient and were chosen because:
- Multi-generation support works via chained/recursive queries (tested against a 3-generation example: grandmother → son & wife → 2 kids).
- Much less data duplication than a bidirectional relationship table (12 rows for 5 people vs. 3 simple columns).
- Directly answers the one relationship question the app is expected to need eventually: **"who are this child's parents?"** — relevant for a future kids'-program drop-off scenario, without building full pickup-authorization machinery now.

**Explicitly deferred (not built yet):** drop-off/pickup verification for kids' programs (pickup codes, authorized-pickup lists, staffed pickup verification) — flagged as a distinct future feature, intentionally out of scope for the initial build.

---

## 8. Explicitly out of scope

- **Payment tracking** — not this app's responsibility. No payment-related columns anywhere in this schema. If an event requires payment, that's presumably enforced upstream by whichever external process manages `event_registrations` — this app only ever checks "does a registration row exist," never why one does or doesn't.
- **In-app volunteer-notification / help-request flow** — considered and deliberately simplified away. No-match and other blocked states just redirect to the front desk with a static message; no push notifications, no volunteer companion app, no "help requested" device state.
- **Kids' program pickup authorization** — see Section 7. Deferred to a later phase.
- **Database-level enforcement of household grouping validity** — see Section 5. No FK, no backing table; the application/registration process is trusted to assign `household_id` consistently.

---

## 9. Platform & technology decisions

| Decision | Choice | Rationale |
|---|---|---|
| Mobile framework | **Flutter** | Single codebase covers phone app + iPad kiosk; Dart's static typing is a smoother transition from a Java/backend background than JS; strong plugin support for camera scanning and kiosk lock mode. React Native was a close second — stronger if volunteer/contractor help from a general JS background becomes likely, since the JS talent pool is significantly larger than Dart's. |
| QR/barcode input | **Two parallel input methods** on the identify screen | Camera scanning via `mobile_scanner` package. External USB/Bluetooth barcode scanners work via **HID/keyboard-wedge mode** — no vendor SDK needed, the device "types" the scanned code + Enter; a `KeyboardListener` widget buffers fast keystrokes and detects the Enter terminator to distinguish a scan from human typing. |
| Backend | **Supabase** (Postgres) | Auto-generated REST/GraphQL layer from the schema; Postgres translates directly from the relational design already built. Row Level Security should be enabled with access only via a service role / backend layer — the personal directory should never be queryable directly from client apps. |
| Local caching | **Selective** | Never cache the personal user/member directory on-device (privacy risk on an unattended public kiosk). Safe to cache: today's active events list (low-sensitivity, slow-changing, ~5 min TTL) and outgoing check-in writes for offline resilience (queued locally, retried on reconnect). Never cache a specific person's data beyond their own check-in session. |
| API call frequency | One call per **completed, deliberate action** | Not per keystroke — phone/email search waits for explicit submission or valid-format completion. Not per camera frame — QR scanning needs debouncing since `onDetect` fires repeatedly on a held code. Hardware scanner input is naturally single-fire (on the Enter terminator). |

---

## 10. Operational reliability & security requirements

### API / trust boundary
Clients never receive direct database credentials with permission to query the member directory. Phone/email/QR resolution and check-in writes go through a trusted backend/API layer. Supabase Row Level Security remains enabled on member and attendance tables; client-visible credentials must not permit arbitrary `users` reads. The exact authentication mechanism may differ between the personal app and kiosk, but both use the same constrained backend operations rather than unrestricted table access.

Minimum backend operations for v1:
- Resolve identity by QR, normalized phone, or normalized email.
- Load the matched person's household (all `users` rows sharing the same `household_id`).
- Load currently eligible events and registrations for the selected household members.
- Submit a completed check-in session atomically/idempotently.

### Offline queue & idempotency
Only completed check-in writes may be queued offline. Each deliberate household/session submission receives one client-generated `client_request_id` UUID before the first network attempt. Every `check_ins` row created by that submission carries the same value, and retries reuse it. The backend treats the request as idempotent: an identical retry returns the existing result rather than creating additional attendance rows. `client_request_id` is indexed but not unique because one submission may legitimately create multiple attendance rows. For v1, persisted idempotency applies to submissions that create at least one attendance row; a zero-row submission has no durable request record unless a separate request/session table is introduced later.

Queued writes contain only the minimum data needed to finish the attendance transaction; they must not contain a cached copy of the broader member directory. If a queued submission is retried after the event's check-in window closes, the backend may accept it when the original client timestamp shows the submission was completed while the event was eligible; otherwise it is rejected for staff review.

### Kiosk privacy reset acceptance criteria
Returning visually to the idle screen is not sufficient. Every success, cancellation, error timeout, or inactivity reset must also clear:
- matched person and household objects from in-memory UI/session state;
- typed phone/email values;
- scanner/barcode buffers and pending scan debounce state;
- selected household members and event selections;
- navigation/history state that could expose the previous household via back navigation;
- transient error/success messages containing member information.

No prior household data may be visible or recoverable through normal kiosk interaction after reset.

---

## 11. Resolved design decisions (chronological log)

- Family check-in: any household member's identifier surfaces the full household checklist, not just themselves.
- No-match handling: simplified to a static front-desk redirect message — no volunteer-notification system.
- Household-level phone/email: removed — ambiguous ownership among multiple members; lookups match individual normalized `users.phone`/`email` instead.
- "Head of household" / primary contact: considered, then removed entirely — not needed by any current feature.
- Family relationships: simple self-referencing columns (`parent_1_id`, `parent_2_id`, `spouse_id`) on `users`, chosen over a dedicated relationships table — sufficient for multi-generation queries without unneeded complexity.
- Multiple concurrent events: supported via auto-fill-from-registration (editable) plus an opt-in per-person flow for split households, avoiding a default matrix UI.
- Registration-required events with no registration: **blocked**, not allowed as walk-ins — redirects to front desk.
- Payment tracking: **out of scope** entirely — not represented anywhere in this schema or app logic.
- `household_name`: removed — served no functional purpose beyond a display label; the checklist screen shows member names directly, with no group heading needed.
- Duplicate check-ins: v1 is idempotent per person + event; repeat scans return `already_checked_in` rather than creating another row. Re-entry counting is deferred.
- Contact lookup normalization: backend/import process maintains normalized phone/email values and lookup indexes use those canonical fields.
- Event eligibility: explicit `checkin_opens_at` / `checkin_closes_at` window, falling back to event start/end only when not supplied.
- Offline retry behavior: each completed household/session submission carries one persistent `client_request_id`; all rows from that submission share it, and retries reuse it.
- Kiosk reset: privacy reset clears both visible UI and all in-memory/session/input/navigation state from the previous household.
- Client/database boundary: clients use constrained backend operations; the member directory is never exposed through unrestricted client-side table access.
- Ambiguous contact lookup: multiple phone/email matches are accepted only when all matches resolve to the same household; cross-household matches return `ambiguous_match` and route to the front desk without exposing member data.
- Check-in window fallback: each boundary resolves independently with `coalesce(checkin_opens_at, start_time)` and `coalesce(checkin_closes_at, end_time)`.
- Idempotency indexing: `client_request_id` is non-unique and indexed for efficient retry lookup; persisted v1 idempotency applies when a submission creates at least one attendance row.
- **`households` table: dropped entirely (Option B).** `household_id` is now a plain shared UUID on `users` with no foreign key and no backing table. Chosen over keeping a minimal enforcement-only table — for this project's single-maintainer scope, the referential-integrity guardrail wasn't judged worth the extra table and registration step. Grouping consistency is now the application/registration process's responsibility, not the database's.

---

## 12. Still open

- Kiosk orientation (portrait vs. landscape).
- Number of kiosks needed for large events (e.g. Pran Pratishtha–scale) and resulting backend concurrency/load-test targets.
- External barcode scanner hardware selection (confirm HID/keyboard-wedge mode before purchase).
- Whether large events need a separate staffed registration counter distinct from the kiosk's front-desk redirect.
- How `event_registrations` data actually reaches this system — synced into this database, or queried live from an external system at check-in time. Needs confirmation with whoever owns that process.
- Final authentication mechanism for each client shell (personal app vs. kiosk) and the exact Supabase RLS policies/backend deployment pattern that implement the trust boundary defined in Section 10.
- How `household_id` values are generated and kept consistent at registration time, now that there's no database-level enforcement backing them.

---

## 13. Reference files

The full database schema is included inline in Section 5 above — copy that SQL block directly into Supabase's SQL editor to run it.

Other related files not embedded here:
- `temple-checkin-kiosk-flow.drawio` — editable flow diagram (identify → household → event → confirm → reset).
- Wiki pages (`Home.md`, `User-Flows.md`, `Kiosk-UX.md`, `Open-Decisions.md`) — public-facing / team-shareable version of this design, kept at a conceptual level without schema/API implementation detail.
