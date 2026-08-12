# AGENTS.md

## Project

This repository contains the Temple Check-In application.

The system consists of:

- Flutter mobile application
- Self-service iPad kiosk experience
- Supabase/PostgreSQL backend
- Trusted backend operations for identity lookup and check-in
- Supabase Row Level Security

---

## Source of Truth

Before making implementation changes, read:

1. `README.md`
2. `docs/master-design.md`

`docs/master-design.md` is the authoritative product and architecture specification.

Do not silently change requirements defined in that document.

If a requested implementation conflicts with the design document:

- identify the conflict;
- explain it clearly;
- prefer the documented design unless the user explicitly requests a change.

Do not invent product behavior when the specification already defines it.

---

## Documentation Synchronization

This file intentionally mirrors implementation-relevant rules from `docs/master-design.md` so coding agents have critical constraints available directly in their working instructions.

`docs/master-design.md` remains the authoritative product and architecture specification.

When a design decision changes in `docs/master-design.md`, review this `AGENTS.md` file for duplicated or related implementation rules and update both documents together where necessary.

Do not allow the two documents to silently diverge.

If `AGENTS.md` and `docs/master-design.md` conflict:

1. prefer `docs/master-design.md`;
2. report the inconsistency;
3. do not silently choose a different behavior.

---

## Implementation Philosophy

Implement the project incrementally.

Prefer small, reviewable changes over broad rewrites.

Do not redesign or re-architect unrelated parts of the application while implementing a specific feature.

When given a task:

1. Read the relevant documentation and existing code.
2. Identify the smallest correct implementation.
3. Make the change.
4. Add or update tests.
5. Run relevant tests and static analysis.
6. Report what changed and any remaining issues.

Do not report a task as complete if relevant tests are failing.

---

## Technology Stack

### Client

- Flutter
- Dart

The same Flutter codebase should support:

- iOS
- Android
- iPad kiosk mode

### Backend

- Supabase
- PostgreSQL
- Supabase migrations
- Supabase Row Level Security
- Trusted backend/API operations

Use repository conventions once they are established.

Do not introduce another primary framework, database, or backend platform unless explicitly requested.

---

## Database Changes

All database schema changes must be implemented through versioned Supabase migrations.

Do not manually modify the expected production schema outside migrations.

Schema changes must be:

- repeatable from a clean database;
- represented in source control;
- consistent with `docs/master-design.md`;
- accompanied by appropriate tests.

Do not modify historical migrations that may already have been applied unless explicitly instructed.

Create a new migration instead.

---

## Member and Household Model

The `users` table is the atomic member record.

Households are represented by a shared:

`household_id`

There is no separate `households` table in the current design.

Do not introduce a `households` table unless the product design is explicitly changed.

Members sharing the same `household_id` belong to the same operational check-in group.

Household grouping integrity is managed by the registration/import process.

Do not derive household membership from family-relationship fields.

---

## Family Relationship Fields

The `users` table may contain:

- `parent_1_id`
- `parent_2_id`
- `spouse_id`

These fields represent factual family relationships and are separate from operational household grouping.

Do not derive `household_id` from these relationship fields.

Existing database constraints must continue to prevent:

- a user being their own `parent_1_id`;
- a user being their own `parent_2_id`;
- a user being their own `spouse_id`;
- the same user being assigned as both `parent_1_id` and `parent_2_id`.

Do not remove or weaken these constraints.

Relationship fields should only be populated when the owning registration/import/admin process has reliable relationship information.

Do not infer family relationships from:

- shared `household_id`;
- shared last name;
- shared address;
- age;
- gender;
- registration order.

`spouse_id` reciprocity is maintained by the owning registration/admin process rather than database triggers unless the design is explicitly changed.

Current check-in functionality does not depend on these relationship fields.

Do not expand relationship logic into sibling inference, household derivation, pickup authorization, or other family-management features unless explicitly requested and reflected in `docs/master-design.md`.

---

## Identity Resolution

Supported identity methods are:

- QR code
- normalized phone number
- normalized email address
- barcode support where defined by the design

Never perform production lookup against raw free-form phone or email values.

Use:

- `phone_normalized`
- `email_normalized`

QR lookup uses the opaque `qr_code` value.

Never expose internal user IDs in QR codes.

### Multiple Lookup Matches

Phone or email lookup may match more than one user.

Required behavior:

- If all matched users share the same `household_id`, treat the lookup as one household match.
- If matches span more than one `household_id`, return an ambiguous-match result.
- For ambiguous matches, do not expose member names or household data.
- Route the user to the front-desk/help flow defined in the design.

Do not arbitrarily choose one matching user.

---

## Client / Database Trust Boundary

The Flutter application must not have unrestricted access to the member directory.

Clients must use constrained backend operations for:

- QR lookup
- phone lookup
- email lookup
- household retrieval
- eligible event retrieval
- registration resolution
- check-in submission

Do not solve application problems by granting clients unrestricted `SELECT` permissions on `users`.

Do not bypass the backend trust boundary for convenience.

---

## Supabase Security

Supabase Row Level Security must remain enabled for sensitive application tables.

Never disable RLS simply to make a feature work.

Never expose the Supabase service-role key in:

- Flutter code
- client configuration
- committed files
- frontend environment variables
- tests bundled into the client
- logs

Service-role credentials may only be used in trusted server-side environments where appropriate.

Never commit secrets.

Use environment variables and documented local-development configuration.

---

## Personally Identifiable Information

Member records contain personal information.

Do not unnecessarily log:

- member names;
- phone numbers;
- email addresses;
- dates of birth;
- QR values;
- entire user objects;
- entire household objects;
- Supabase credentials;
- authentication tokens.

Diagnostic logging should use non-sensitive identifiers where possible.

Never include personal member data in exception messages returned to unrelated users.

---

## Event Eligibility

Check-in eligibility is based on effective event check-in windows.

Use:

`effective_checkin_opens_at = coalesce(checkin_opens_at, start_time)`

and:

`effective_checkin_closes_at = coalesce(checkin_closes_at, end_time)`

An event is eligible only when the current time falls within its effective check-in window.

Do not use a vague calendar-day or `"today"` rule to determine eligibility.

---

## Registration Rules

For events where:

`requires_registration = true`

a member must have a matching `event_registrations` row.

If no registration exists:

- do not create attendance;
- return the registration-required/blocked result defined by the product design.

Registration eligibility is evaluated per person and per event.

Do not infer registration from another household member.

---

## Check-In Rules

Attendance is stored one row per:

`user_id + event_id`

The database must enforce this uniqueness.

A repeat check-in must not create another attendance row.

Return:

`already_checked_in`

or the established equivalent instead.

Re-entry counting is out of scope for v1 unless the design document is explicitly changed.

---

## Household Check-In Sessions

One household interaction may create multiple attendance rows.

Rows created by the same interaction share:

- `checkin_session_id`
- `client_request_id`

`client_request_id` must not be globally unique on individual `check_ins` rows because multiple rows intentionally share it.

It should be indexed for retry/idempotency lookup.

---

## Idempotency

A completed check-in submission must be safe to retry.

Generate one `client_request_id` for the deliberate submission before the first network attempt.

Retries must reuse the same ID.

The backend must avoid creating duplicate attendance records when the same request is retried.

Idempotency applies to submissions that create attendance records unless a future request-tracking mechanism explicitly extends this behavior.

Never generate a new request ID merely because a network retry occurred.

---

## Transactions

Household check-in submission should be transactional where practical.

Unexpected server or database failures must not silently leave an inconsistent partial submission.

Expected business outcomes such as:

- `already_checked_in`
- `registration_required`
- `event_not_eligible`
- `ambiguous_match`
- `not_found`

must be handled explicitly rather than treated as generic server failures.

---

## Offline Behavior

The application may queue completed check-in submissions for retry.

Do not cache the full member directory for offline use.

Offline queued records should contain only the minimum information required to complete the intended attendance submission.

Retries must retain the original:

`client_request_id`

Do not weaken privacy protections to support offline functionality.

If a queued request is retried after an event window closes, follow the behavior defined in `docs/master-design.md`.

---

## Kiosk Privacy

Kiosk reset is a security requirement, not just navigation behavior.

After:

- successful check-in;
- cancellation;
- inactivity timeout;
- error timeout;
- explicit reset;

clear all prior household session information.

This includes:

- identified person;
- household/member objects;
- phone/email input;
- scanner buffers;
- barcode input;
- debounce state;
- selected members;
- selected events;
- transient messages containing personal data;
- navigation history that could expose previous screens.

After reset, the next kiosk user must not be able to recover the previous household's information through normal application interaction.

Tests should verify this behavior.

---

## QR and Barcode Scanning

QR detection must be debounced.

Camera scanners can produce repeated detection callbacks while a code remains visible.

One physical scan should not trigger multiple lookup or check-in operations.

Hardware barcode scanners operating as keyboard/HID input should be treated as a discrete scan after their configured terminator, typically Enter.

Do not introduce vendor-specific scanner dependencies unless required.

---

## Flutter Architecture

Keep feature boundaries clear.

Prefer an organization similar to:

```text
lib/
  core/
  data/
  domain/
  features/
    identify/
    household/
    event_selection/
    checkin/
  kiosk/
```

Follow established project conventions if the actual structure evolves.

Avoid placing:

- database access;
- business rules;
- UI state;
- scanner integration;

all inside large widget files.

Business rules should be independently testable from UI rendering.

---

## Backend Operations

Prefer explicit, constrained backend operations rather than generic direct table access.

Expected operations will likely include equivalents of:

- `resolveIdentity`
- `getHousehold`
- `getEligibleEvents`
- `getRegistrations`
- `submitCheckinSession`

Exact naming and implementation mechanism may evolve.

Backend operations must enforce product rules server-side.

Do not rely exclusively on Flutter UI validation for:

- event eligibility;
- registration requirements;
- duplicate attendance prevention;
- lookup ambiguity;
- authorization.

---

## Error Handling

Expected application states should use explicit typed or structured results where practical.

Examples include:

- `matched`
- `not_found`
- `ambiguous_match`
- `already_checked_in`
- `registration_required`
- `event_not_eligible`
- `validation_error`

Do not use generic exceptions for normal business outcomes.

Unexpected errors should be logged without exposing personal data.

---

## Testing Requirements

Add tests for important behavior introduced or modified by a task.

### Backend / Database Tests

Cover relevant cases such as:

- QR lookup
- normalized phone lookup
- normalized email lookup
- multiple matches within one household
- ambiguous matches across households
- registration-required events
- unrestricted events
- event eligibility windows
- duplicate check-in
- request retry/idempotency
- concurrent duplicate submissions
- relationship self-reference constraints
- duplicate-parent constraints

### Flutter Tests

Cover relevant states such as:

- identification success
- no match
- ambiguous match
- household selection
- event selection
- already checked in
- registration blocked
- kiosk reset
- timeout behavior

Do not remove valid tests simply to make a change pass.

---

## Validation Before Completion

For Flutter changes, run relevant available commands, normally including:

```bash
flutter analyze
flutter test
```

For Supabase/database changes, run the project's established local migration and test workflow.

If Supabase CLI is configured, verify migrations against a clean local database where practical.

If a validation command cannot be run, say so explicitly in the final response.

Do not claim tests passed unless they were actually executed successfully.

---

## Dependencies

Avoid adding dependencies unless they materially simplify or improve the implementation.

Before adding a new package:

1. Confirm existing dependencies do not already solve the problem.
2. Prefer mature and actively maintained packages.
3. Keep the dependency surface small.
4. Explain why the new dependency is needed.

Do not replace established packages without a clear technical reason.

---

## Scope Control

Do not implement features explicitly listed as out of scope in `docs/master-design.md`.

Examples currently include:

- payment tracking;
- volunteer notification/help-request systems;
- kids' pickup authorization;
- re-entry counting.

If a task appears to require an out-of-scope feature, identify that conflict before expanding the implementation.

---

## Documentation

Use documentation according to its purpose:

- `README.md` — project orientation, setup, and high-level overview
- `docs/master-design.md` — authoritative product and architecture requirements
- `AGENTS.md` — durable implementation rules for coding agents
- code comments — non-obvious implementation reasoning
- migrations — database evolution

Do not copy excessive implementation detail into `README.md`.

When implementation establishes an important architectural decision not already documented, update the appropriate documentation.

If a change affects a rule duplicated in both `docs/master-design.md` and `AGENTS.md`, review and update both together.

---

## Git and Repository Hygiene

Do not commit:

- secrets;
- API keys;
- service-role credentials;
- generated build artifacts;
- temporary debugging files;
- local environment files containing credentials;
- local IDE configuration unless intentionally shared.

Respect `.gitignore`.

Avoid unrelated formatting or refactoring in feature-specific changes.

Do not rewrite Git history unless explicitly requested.

---

## When Requirements Are Unclear

First inspect:

- `docs/master-design.md`
- existing code
- existing tests
- database migrations
- relevant repository documentation

Prefer consistency with the documented design and existing implementation.

If a material product decision is genuinely unspecified, surface the ambiguity rather than inventing major new behavior.

Small implementation details may be resolved using the simplest secure approach consistent with the existing architecture.

---

## Definition of Done

A task is complete when:

- the requested behavior is implemented;
- implementation follows `docs/master-design.md`;
- security boundaries remain intact;
- required migrations are present;
- relevant tests have been added or updated;
- relevant tests pass;
- static analysis passes where applicable;
- no secrets or unnecessary personal data are introduced;
- documentation is updated when needed.

At completion, provide a concise report containing:

- what changed;
- tests and validation run;
- files materially changed;
- unresolved issues;
- assumptions made.