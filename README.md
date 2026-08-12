# Temple Check-In

Temple Check-In is a mobile and kiosk application for checking temple members and households into events.

The system supports:

* iOS and Android personal mobile use
* Self-service iPad kiosk check-in
* QR-code identification
* Phone and email lookup
* Household/family check-in in a single interaction
* Pre-registered attendees
* Walk-in attendees where permitted
* Multiple concurrent events
* Registration-required event enforcement
* Duplicate check-in protection
* Offline-safe check-in submission
* Privacy-focused kiosk session reset

## Technology Stack

### Client

* Flutter
* Dart
* iOS
* Android
* iPad kiosk mode

### Backend

* Supabase
* PostgreSQL
* Supabase Row Level Security
* Trusted backend/API operations for member lookup and check-in

## Architecture

The core application flow is:

```text
Identify
   ↓
Resolve Household
   ↓
Select Who Is Present
   ↓
Resolve Eligible Events
   ↓
Confirm
   ↓
Submit Check-In
```

The same core workflow is used by both the personal mobile application and the self-service kiosk.

The kiosk adds additional requirements including inactivity timeout, session clearing, privacy reset, and device lockdown.

## Authoritative Design Specification

The complete product and technical specification is located at:

```text
docs/master-design.md
```

**The master design document is the authoritative specification for this project.**

Developers and coding agents should read it before making architectural, database, security, or product-behavior changes.

If implementation behavior conflicts with the master design document, the discrepancy should be identified and resolved explicitly rather than silently changing the intended behavior.

## Major Design Principles

### Household Check-In

A household is an operational grouping of people who normally check in together.

Members of the same household share a `household_id`.

Scanning or looking up any household member loads the entire household so the user can select who is present.

### Identity Resolution

Supported identification methods are:

* QR code
* Phone number
* Email address
* Physical QR/barcode card in a future phase

Phone and email searches use normalized backend values rather than raw display values.

Identity resolution must happen through constrained backend operations.

The Flutter application must not be given unrestricted access to query the member directory.

### Events

Events may:

* Allow walk-ins
* Require prior registration
* Have explicit check-in opening and closing times
* Run concurrently with other events

Registration-required events cannot be selected by a person without an existing registration.

### Check-In

Attendance is recorded per:

```text
user + event
```

A person cannot create duplicate attendance records for the same event.

Repeated attempts return an `already_checked_in` result rather than creating another record.

A household submission may create multiple attendance rows sharing the same:

```text
checkin_session_id
client_request_id
```

### Security

Member information is considered sensitive application data.

Important security requirements include:

* Supabase Row Level Security remains enabled.
* Flutter clients do not receive service-role credentials.
* Clients cannot perform arbitrary reads against the `users` table.
* Identity lookup occurs through trusted backend operations.
* Personal data should not be written to application logs.
* Kiosk sessions must completely clear previous household state after completion, cancellation, timeout, or error.

## Repository Structure

```text
.
├── README.md
├── AGENTS.md
├── docs/
│   └── master-design.md
├── app/
│   ├── lib/
│   └── test/
├── supabase/
│   ├── migrations/
│   ├── functions/
│   ├── tests/
│   ├── seed.sql
│   └── config.toml
└── .gitignore
```

The repository structure may evolve as implementation progresses, but architectural changes must remain consistent with `docs/master-design.md`.

## Development Approach

Implementation should proceed incrementally.

Recommended order:

1. Supabase project initialization
2. Database migrations
3. Seed/test data
4. Row Level Security policies
5. Backend identity-resolution operations
6. Event and registration operations
7. Check-in transaction and idempotency
8. Flutter project setup
9. Identity flow
10. Household selection
11. Event selection
12. Check-in confirmation
13. Kiosk workflow
14. Offline queue
15. Integration and security testing

Avoid attempting to implement the entire application as one large change.

Each feature should include appropriate tests before moving to the next phase.

## Development Rules

* Read `docs/master-design.md` before implementing a feature.
* Database changes must be made through migrations.
* Do not manually modify production database schema.
* Do not disable RLS to solve application problems.
* Do not expose Supabase service-role credentials to the client.
* Do not allow unrestricted client-side queries of member data.
* Do not introduce functionality explicitly listed as out of scope in the design document.
* Do not silently change product requirements.
* Add automated tests for important backend and client behavior.
* Keep personally identifiable information out of logs.
* Run relevant tests before completing a change.

## Current Status

The project is currently in the initial implementation phase.

The product architecture and baseline requirements have been defined in:

```text
docs/master-design.md
```

Implementation should follow that specification.

## Documentation

Primary documentation:

```text
docs/master-design.md
```

Additional implementation documentation may be added under:

```text
docs/
```

as the project evolves.

## License

To be determined.

