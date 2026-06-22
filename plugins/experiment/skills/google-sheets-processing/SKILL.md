---
name: google-sheets-processing
description: >-
  Guides implementation of Google Sheets processing assets — the SurgeryLog / WaterLog classes that
  read animal and session records into typed platform data and write results back. Covers the
  processor API, service-account auth, the sheet schema contract, how an acquisition system wires
  sheets, and authoring a custom processor. Use when reading or writing Google Sheets data,
  adapting a processor to a new sheet schema, or wiring sheets into an acquisition system.
user-invocable: false
---

# Google Sheets processing

Guides the implementation of Google Sheets **processing assets** — the source components that read
records out of a Google Sheet into typed platform data and write runtime results back. This is the
platform-general Google Sheets subsystem: the `SurgeryLog` / `WaterLog` stack in
`sollertia_experiment/cross_system/google_sheet_tools.py` is system-agnostic (it lives in
`cross_system/` and is exported from `cross_system/__init__.py`) and consumed by any acquisition
system's preprocessing layer. Mesoscope-VR is the current consumer — its
`_preprocess_google_sheet_data` (in `mesoscope_vr/data_preprocessing.py`) and the shared
`snapshot_surgery_data` (in `cross_system/data_preprocessing.py`) instantiate these processors.

A **processing asset** is an external-service client. It is constructed
directly by preprocessing / per-session setup code on demand, standing outside the Layer-2b
`start`/`stop` surface and the slsa registry (the read-asset dataclass a read processor produces
IS a registered slsa asset — see below). For where this category
sits in the acquisition-system architecture, see `/acquisition-system-design`
([references/subsystem-types.md](../acquisition-system-design/references/subsystem-types.md),
"External data-service processors").

---

## Scope

**Covers:**
- The `SurgeryLog` and `WaterLog` processor API (constructors, extract / update methods, teardown)
- The service-account authentication and connection model (scopes, retries, `__del__` cleanup)
- The sheet schema contract each processor validates at construction (required headers, tab layout,
  identity model) — see [references/sheet-schema-contract.md](references/sheet-schema-contract.md)
- How an acquisition system wires sheet identifiers and gates processing on them
- Authoring a custom processor for a sheet whose schema differs from the platform default, or for a
  novel acquisition system

**Does not cover:**
- The system-configuration field that stores the sheet identifiers (`surgery_sheet_id`,
  `water_log_sheet_id`) — owned by the per-system instance skill (`mesoscope:mesoscope-vr`) and the
  external-services auxiliary section in `/acquisition-system-design`
- Setting the credentials-file path on the host — owned by `assets:working-directory`
- Reading or amending the `surgery_metadata.yaml` snapshot produced by `SurgeryLog` — owned by the
  `assets:data-assets`
- The `SurgeryData` / `SubjectData` / `ProcedureData` / `DrugData` / `ImplantData` / `InjectionData`
  dataclasses themselves (owned by `sollertia-shared-assets`; extend via assets `assets:library-extension`)
- The session-lifecycle tools that invoke preprocessing (`/data-management`)
- The `googleapiclient` / `google.oauth2` library internals (third-party; consult their own docs)

---

## When to use this skill

- Reading animal surgery records or water-restriction logs out of a Google Sheet
- Writing a surgery-quality score or a session's water-restriction summary back to a sheet
- Adapting `SurgeryLog` / `WaterLog` to a sheet whose column or tab layout differs from the default
- Authoring a brand-new processor for a different external log when adopting the platform
- Diagnosing a preprocessing failure that traces to sheet parsing (missing headers, animal not in
  the sheet, a date row that is not pre-filled)

---

## Prerequisites

A processor needs three things before it can connect:

| Prerequisite             | Where it comes from                                                                                                                                                                                                             |
|--------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Credentials file path    | A Google Cloud **service-account** JSON key. The host path is set via assets `assets:working-directory` and resolved at call time by `get_credentials(credentials=CredentialsTypes.GOOGLE)`.                                    |
| Sheet identifier         | The long alphanumeric segment of the sheet URL, stored in the system configuration's external-services section (`surgery_sheet_id` / `water_log_sheet_id` for Mesoscope-VR).                                                    |
| Sheet shared with the SA | The target Google Sheet MUST be shared with the service account's email (as a viewer for read-only, as an editor for any processor that writes). Google service accounts have no access until the document is shared with them. |

Both processors request the full `https://www.googleapis.com/auth/spreadsheets` scope (read **and**
write), because both write back — `SurgeryLog` updates the surgery-quality cell and `WaterLog`
updates the session row.

---

## The two processing assets

Both classes are constructed with `(identity…, credentials_path, sheet_id)`, validate the sheet's
shape in `__init__` (raising `ValueError` via `console.error` on any mismatch), expose
extract / update methods, and close the HTTP connection in `__del__`.

### SurgeryLog — read the surgery record, write the quality score

```text
SurgeryLog(project_name: str, animal_id: int, credentials_path: Path, sheet_id: str)
```

| Member                                 | Purpose                                                                                                                                                               |
|----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `extract_animal_data()`                | Parses the animal's row into a `SurgeryData` instance (subject, procedure, drugs[], implants[], injections[]).                                                        |
| `update_surgery_quality(quality: int)` | Writes the surgery-quality score into the animal's row and applies center/middle cell alignment. The 0–3 scale is advisory; the value is written exactly as provided. |
| `__del__`                              | Closes the HTTP service.                                                                                                                                              |

**Identity model:** one **tab per project** (tab name = `project_name`); header row = **row 1**;
animals are rows 2+, keyed by a zero-padded five-digit value in the **`id` column**. Construction
fails if the project tab's header row is empty, a required header is missing, the `id` column is
empty, or the target `animal_id` is absent from it.

### WaterLog — write a session's water-restriction summary

```text
WaterLog(animal_id: int, session_date: str, credentials_path: Path, sheet_id: str)
```

`session_date` is the Sollertia session name (`YYYY-MM-DD-HH-MM-SS-US`); it is parsed to local time
and used to locate the pre-filled date row for this session.

| Member                                                              | Purpose                                                                                                                       |
|---------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| `update_water_log(weight, water_ml, experimenter_id, session_type)` | Writes the five session cells (weight, given-by, water given, behavior, time) into the resolved session row, with formatting. |
| `__del__`                                                           | Closes the HTTP service.                                                                                                      |

**Identity model:** one **tab per animal** (tab name = the numeric animal ID); header row = **row
2** (the surgery log uses row 1); data rows are 3+, with the **date column
pre-filled**. Construction fails if no digit-named animal tabs exist, the target animal's tab is
missing, a required header is absent, the session name is not a valid timestamp, or the session's
date row is not already present in the sheet.

The full required-header sets, the dynamic implant/injection column convention, the coordinate and
date formats, and the `SurgeryData` field-to-header mapping live in
[references/sheet-schema-contract.md](references/sheet-schema-contract.md).

---

## The processing-asset contract

Every processor — the two above and any custom one — satisfies the same contract. This is what makes
the category recognizable and what a custom processor must reproduce:

1. **Construction validates, then caches.** The constructor authenticates, fetches the header row,
   builds a `header → column-letter` map, and asserts every required header is present and the target
   record exists. A malformed sheet fails **at construction**, before any extract/update call,
   so every parse problem surfaces as a construction error.
2. **Authentication is service-account based.** `Credentials.from_service_account_file(scopes=…)`
   builds a `sheets`/`v4` service with `cache_discovery=False` (the discovery cache is unsupported by
   the installed client and only emits a spurious warning).
3. **Every API call is retried.** All `.execute()` calls pass `num_retries=_GOOGLE_API_MAX_RETRIES`
   (5) so transient 5xx/429 errors self-heal.
4. **The connection is owned and released.** The service handle is a private attribute; `__del__`
   closes it. A processor handle may be returned to the caller so a follow-up write reuses the open
   connection (e.g., `snapshot_surgery_data` returns the `SurgeryLog` so the runtime can later call
   `update_surgery_quality`).
5. **The processor is invoked directly.** Preprocessing / per-session setup constructs it on demand,
   standing outside DataLogger orchestration, the `start`/`stop` surface, the processor registry, and
   import-time parity checks. (The read-asset *dataclass* it produces is separately registered in
   slsa's `READ_ASSET_REGISTRY`; see the authoring section below.)

---

## How an acquisition system wires Google Sheets

The processors are pure consumers of two inputs the system supplies — a credentials path and a sheet
identifier. The system configuration's **external-services** auxiliary section holds the identifiers
(for Mesoscope-VR, `MesoscopeGoogleSheets.surgery_sheet_id` and `.water_log_sheet_id`); the
host-level credentials path is resolved separately. Preprocessing then constructs the processors:

- `snapshot_surgery_data(session_data, animal_id, credentials_path, surgery_sheet_id)` builds a
  `SurgeryLog`, calls `extract_animal_data()`, writes the result to the session's
  `surgery_metadata.yaml`, and returns the handle.
- `_preprocess_google_sheet_data(session_data, sheets_data)` is the Mesoscope-VR glue: it gates the
  whole step on the configured identifiers, resolves credentials, snapshots the surgery record, and
  updates the water log.

**Gating rules** (mirror these in any system):

- If **neither** identifier is set, all sheet processing is skipped with a warning and **no
  credentials are required** — this is how a system opts out of Google Sheets entirely.
- If **at least one** identifier is set, the host MUST provide valid credentials; a missing or
  invalid credentials file aborts preprocessing with `FileNotFoundError`.
- A sheet whose identifier is individually unset (while the other is set) is skipped with a warning.

---

## Authoring a custom processor

The shipped processors are hard-coded to the Sollertia sheet schema (their docstrings say
"purpose-built to work with the specific format used by the Sollertia platform"). System-agnostic
means **any acquisition system can consume them** while they stay bound to the single Sollertia
sheet schema. So:

- **Sheet matches the Sollertia schema** → reuse `SurgeryLog` / `WaterLog` as-is. Just set the sheet
  identifiers in the system configuration and share the document with the service account. No code.
- **Sheet schema differs, or the source is a different service** (a lab's own water-log layout, a
  LIMS, a REST registry) → author a new processor that reproduces the contract above against the new
  schema. Decide the **direction** up front:
  - A **read processor** (like `SurgeryLog`) parses records into a typed platform dataclass that is
    snapshotted to disk — a **registered read asset** in slsa (`SurgeryData` is the `surgery_data`
    read asset). This standardizes the downstream (slf) interface: forging reads only the on-disk
    dataclass. If the processor emits a *new* record type rather than
    `SurgeryData`, add the dataclass + its `ReadAssets` member + `READ_ASSET_REGISTRY` entry through the
    `assets:library-extension`'s "Adding a new read asset" scenario, and its read/amend surface
    through `assets:data-assets`.
  - A **write processor** (like `WaterLog`) consumes runtime-discovered values and writes them into a
    pre-existing row/record.

The step-by-step procedure — schema-contract definition, the read/write split, lifecycle
implementation, placement (`cross_system/` vs the system package), wiring, and documentation — is
the **"Authoring a custom data-service processor"** workflow in
[`/acquisition-system-design` → references/workflows.md](../acquisition-system-design/references/workflows.md).
This skill owns the *what* (the API and contract); that workflow owns the *how* (the procedure).

---

## Error model

Processors report failures through `console.error(message=…, error=ValueError)`, which logs and
raises. Construction is atomic: a processor either constructs cleanly or aborts.

| Symptom                                              | Cause                                                                       |
|------------------------------------------------------|-----------------------------------------------------------------------------|
| `ValueError` naming missing headers                  | The sheet's header row lacks a required column — schema drift or wrong tab. |
| `ValueError`: animal not in the `id` column / no tab | The target animal has no surgery row / no water-log tab.                    |
| `ValueError`: empty header or ID column              | The tab is empty or points at the wrong project/animal.                     |
| `ValueError`: date row not found (`WaterLog`)        | The session's date is not pre-filled in the water log — add it and re-run.  |
| `ValueError`: invalid session timestamp              | The `session_date` passed to `WaterLog` is not a valid session name.        |
| `FileNotFoundError` during preprocessing             | A sheet identifier is set but credentials are missing or invalid.           |
| Malformed cell on extract (`float()`/`int()`/date)   | A weight, cage, date, or time cell is empty or non-numeric.                 |

A migration or preprocessing run surfaces these as the operation's failure (e.g.,
`migrate_animal_tool` fails "when the animal is absent from the surgery sheet"); resolve the sheet
and re-run. See `/data-management` for the lifecycle-level handling.

---

## Related skills

| Skill                                 | Relationship                                                                                                            |
|---------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| `/acquisition-system-design`          | Platform-general home of the "External data-service processors" category and the authoring workflow.                    |
| `mesoscope:mesoscope-vr`              | Current consumer — defines the `MesoscopeGoogleSheets` identifiers these processors read.                               |
| `/data-management`                    | Owns preprocessing, which constructs the processors and updates the sheets.                                             |
| `assets:working-directory`            | Sets and resolves the service-account credentials path the processors require.                                          |
| `assets:data-assets`                  | Reads/amends the on-disk read asset (e.g., the surgery snapshot `SurgeryLog` produces).                                 |
| `assets:library-extension`            | Owns the read-asset registry; register a read processor's emitted dataclass via its "Adding a new read asset" scenario. |
| `/experiment-mcp-environment-setup`   | Run first if the `sle mcp` server is not connected.                                                                     |
| `references/sheet-schema-contract.md` | The full required-header sets, structural assumptions, and field mappings.                                              |

---

## Verification checklist

```text
Before relying on or authoring a Google Sheets processor:
- [ ] Service-account credentials path is set on the host (assets assets:working-directory)
- [ ] The target sheet is shared with the service-account email (editor access if the processor writes)
- [ ] The sheet identifier(s) are set in the system configuration's external-services section
- [ ] The sheet matches the schema contract (required headers, header-row position, identity model)
- [ ] Gating is honored: no credentials required when all identifiers are unset; required when any is set

When authoring a custom processor:
- [ ] Constructor authenticates, builds the header→column map, and validates required headers + record presence
- [ ] Direction is explicit (read → typed dataclass snapshot, and/or write → existing row)
- [ ] A new emitted record type is coordinated via assets assets:library-extension and assets:data-assets
- [ ] Every API call passes num_retries; __del__ closes the connection
- [ ] Placed in cross_system/ if system-agnostic, or the system package if system-specific
- [ ] Wired into the system's preprocessing with the unset-identifier skip / credentials-required gating
- [ ] Documented here (or a sibling skill) and in the consuming system's instance skill
```
