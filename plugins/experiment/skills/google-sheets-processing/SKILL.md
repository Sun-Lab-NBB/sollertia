---
name: google-sheets-processing
description: >-
  Guides implementation of Google Sheets processing assets, the SurgeryLog and WaterLog classes that read
  animal and session records into typed platform data and write results back. Covers the processor API,
  service-account auth, the sheet schema contract, how an acquisition system wires sheets, and authoring a
  custom processor. Use when reading or writing Google Sheets data, adapting a processor to a new sheet
  schema, or wiring sheets into an acquisition system.
user-invocable: false
---

# Google Sheets processing

Guides the implementation of Google Sheets **processing assets**, the source components that read records out of a
Google Sheet into typed platform data and write runtime results back. The `SurgeryLog` and `WaterLog` stack in
`cross_system/google_sheet_tools.py` is system-agnostic, is exported from `cross_system/__init__.py`, and is consumed
by any acquisition system's preprocessing layer through the shared `snapshot_surgery_data` helper in
`cross_system/data_preprocessing.py`.

A **processing asset** is an external-service client that preprocessing or per-session setup constructs directly on
demand, outside the Layer-2b `start`/`stop` binding-class surface. For where this category sits in the
acquisition-system architecture, see `/acquisition-system-design` (its "External data-service processors"
subsystem type).

---

## Scope

**Covers:**
- The `SurgeryLog` and `WaterLog` processor API (constructors, extract / update methods, teardown)
- The service-account authentication and connection model (scopes, retries, `close()` and `__del__` teardown)
- The sheet schema contract each processor validates at construction (required headers, tab layout,
  identity model), detailed in [references/sheet-schema-contract.md](references/sheet-schema-contract.md)
- How an acquisition system wires sheet identifiers and gates processing on them
- Authoring a custom processor for a sheet whose schema differs from the platform default, or for a
  novel acquisition system

**Does not cover:**
- The system-configuration fields that store the sheet identifiers, owned by the per-system instance skill
  (`mesoscope:mesoscope-vr`) and by the external-services auxiliary section in `/acquisition-system-design`
- The session-type branch policy a system applies when it writes back, owned by that system's runtime skill
  (`mesoscope:mesoscope-vr-runtime`)
- Setting the credentials-file path on the host, owned by `assets:working-directory`
- Reading or amending the `surgery_metadata.yaml` snapshot `SurgeryLog` produces, owned by `assets:data-assets`
- The `SurgeryData` / `SubjectData` / `ProcedureData` / `DrugData` / `ImplantData` / `InjectionData`
  dataclasses themselves, owned by sollertia-shared-assets and extended through `assets:library-extension`
- The session-lifecycle tools that invoke preprocessing (`/data-management`)
- The `googleapiclient` and `google.oauth2` library internals, which are third-party and carry their own docs

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

| Prerequisite             | Where it comes from                                                                                                                                                                                           |
|--------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Credentials file path    | A Google Cloud **service-account** JSON key. `assets:working-directory` sets the host path, and `get_credentials(credentials=CredentialsTypes.GOOGLE)` resolves it to `google_credentials.json` at call time. |
| Sheet identifier         | The long alphanumeric segment of the sheet URL, stored in the system configuration's external-services section.                                                                                               |
| Sheet shared with the SA | The target Google Sheet MUST be shared with the service account's email, as a viewer for read-only access and as an editor for any processor that writes.                                                     |

Google service accounts hold no access until the document is shared with them. Both processors request the same
read-write spreadsheets scope, because both write back. `SurgeryLog` updates the surgery-quality cell and `WaterLog`
updates the session row.

---

## The two processing assets

Both classes are constructed with `(identity…, credentials_path, sheet_id)`, validate the sheet's
shape in `__init__` (raising `ValueError` via `console.error` on any mismatch), expose
extract / update methods, and release the HTTP connection through a public `close()` method, with
`__del__` as a garbage-collection backstop.

### SurgeryLog, read the surgery record and write the quality score

```text
SurgeryLog(project_name: str, animal_id: int, credentials_path: Path, sheet_id: str)
```

| Member                                 | Purpose                                                                                                              |
|----------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| `extract_animal_data()`                | Parses the animal's row into a `SurgeryData` instance (subject, procedure, drugs[], implants[], injections[]).       |
| `update_surgery_quality(quality: int)` | Writes the surgery-quality score into the animal's row and applies center/middle cell alignment.                     |
| `close()`                              | Closes the HTTP service. Callers should invoke this from a `try`/`finally` as soon as they finish with the instance. |
| `__del__`                              | Garbage-collection backstop that delegates to `close()`.                                                             |

`update_surgery_quality` documents a 0-3 scale, from 0 for unusable to 3 for publication grade. That scale is
advisory, and the writer validates neither the range nor the type before it writes the cell
(`cross_system/google_sheet_tools.py`).

**Identity model:** one **tab per project** (tab name = `project_name`), header row = **row 1**,
animals in rows 2 and below, keyed by a zero-padded five-digit value in the **`id` column**. Construction
fails when the project tab's header row is empty, a required header is missing, the `id` column is
empty, or the target `animal_id` is absent from it.

### WaterLog, write a session's water-restriction summary

```text
WaterLog(animal_id: int, session_date: str, credentials_path: Path, sheet_id: str)
```

`session_date` is the Sollertia session name (`YYYY-MM-DD-HH-MM-SS-US`). The constructor parses it to local time and
uses it to locate the pre-filled date row for this session.

| Member                                                              | Purpose                                                                                                                       |
|---------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| `update_water_log(weight, water_ml, experimenter_id, session_type)` | Writes the five session cells (weight, given-by, water given, behavior, time) into the resolved session row, with formatting. |
| `close()`                                                           | Closes the HTTP service. Callers should invoke this from a `try`/`finally` as soon as they finish with the instance.          |
| `__del__`                                                           | Garbage-collection backstop that delegates to `close()`.                                                                      |

**Identity model:** one **tab per animal** (tab name = the animal's digits), header row = **row 2**, data rows 3 and
below, with the **date column pre-filled**. Construction fails when no digit-named animal tabs exist, the target
animal's tab is missing, a required header is absent, the session name is not a valid timestamp, or the session's
date row is not already present. `_find_date_row` scans the date column and raises with an instruction to add the
date and rerun the command that failed (`cross_system/google_sheet_tools.py`).

The full required-header sets, the dynamic implant/injection column convention, the coordinate and date formats, and the
`SurgeryData` field-to-header mapping live in
[references/sheet-schema-contract.md](references/sheet-schema-contract.md).

---

## The processing-asset contract

Every processor, the two above and any custom one, satisfies the same contract. This is what makes the category
recognizable and what a custom processor must reproduce:

1. **Construction validates, then caches.** The constructor authenticates, fetches the header row, builds a
   `header → column-letter` map, and asserts every required header is present and the target record exists. A malformed
   sheet *shape* fails **at construction**, before any extract or update call. Cell-*value* parse problems are not
   caught there, because `extract_animal_data` raises `ValueError` from the `int()`, `float()`, and
   `_convert_date_time_to_timestamp` conversions when a row's cells are empty or malformed.
2. **Authentication is service-account based.** `Credentials.from_service_account_file` is scoped to
   `https://www.googleapis.com/auth/spreadsheets` and builds a `sheets`/`v4` service with
   `cache_discovery=False`, because the discovery cache is unsupported by the installed oauth2client version
   and only emits a spurious warning. Both `SurgeryLog.__init__` and `WaterLog.__init__` in
   `cross_system/google_sheet_tools.py` carry this call.
3. **Every API call is retried.** All `.execute()` calls pass `num_retries=_GOOGLE_API_MAX_RETRIES`, which is 5, so
   transient 5xx and 429 errors self-heal (`cross_system/google_sheet_tools.py`).
4. **The connection is owned and released.** The service handle is a private attribute released by the public
   `close()` method, which callers should invoke from a `try`/`finally` as soon as they finish with the instance.
   `close()` first drains the google-auth Regional Access Boundary worker with a 5.0 s join, closes that worker's
   otherwise unreachable transport only once the worker is dead, and then closes the service. `__del__` is a
   backstop only. The `close()` of `SurgeryLog` and the `close()` of `WaterLog` both delegate to the module-level
   `_close_google_service` helper in `cross_system/google_sheet_tools.py`.
5. **A handle may outlive its first use.** `snapshot_surgery_data` returns the open `SurgeryLog` and the caller owns
   it, so the caller MUST `close()` it in a `try`/`finally` to release the SSL socket
   (`cross_system/data_preprocessing.py`).
6. **The processor is invoked directly.** Preprocessing or per-session setup constructs it on demand, so it takes no
   `DataLogger`, joins no `start`/`stop` surface, and appears in no slsa registry. The read-asset *dataclass* it
   produces is registered separately in slsa's `READ_ASSET_REGISTRY`
   (`sollertia-shared-assets/src/sollertia_shared_assets/registries.py`).

---

## How an acquisition system wires Google Sheets

The processors consume two inputs the system supplies, a credentials path and a sheet identifier. The system
configuration's external-services auxiliary section holds the identifiers, and the host-level credentials path
resolves separately through the platform credentials directory.

A system's preprocessing layer then performs three steps. It resolves the credentials once, it snapshots the surgery
record through `snapshot_surgery_data`, and it branches on the session type to pick the write it performs. The branch
policy belongs to that system. For the current worked example, see `mesoscope:mesoscope-vr-runtime`.

**Gating rules** (mirror these in any system):

- A step whose identifiers are all unset is skipped with a warning and needs no credentials, which is how a system
  opts out of Google Sheets entirely.
- A step with at least one identifier set requires host credentials, and an unset credentials file aborts
  preprocessing with the `FileNotFoundError` that `get_credentials` raises
  (`sollertia-shared-assets/src/sollertia_shared_assets/credentials.py`).
- A sheet whose own identifier is unset is skipped with a warning while the other sheet still processes.

---

## Authoring a custom processor

The shipped processors are hard-coded to the Sollertia sheet schema, as their docstrings state that each one is
"purpose-built to work with the specific format used by the Sollertia platform". System-agnostic means **any
acquisition system can consume them** while they stay bound to the single Sollertia sheet schema. So:

- **Sheet matches the Sollertia schema** → reuse `SurgeryLog` / `WaterLog` as-is. Set the sheet
  identifiers in the system configuration and share the document with the service account. No code.
- **Sheet schema differs, or the source is a different service** (a lab's own water-log layout, a
  LIMS, a REST registry) → author a new processor that reproduces the contract above against the new
  schema. Decide the **direction** up front:
  - A **read processor** (like `SurgeryLog`) parses records into a typed platform dataclass that is
    snapshotted to disk as a **registered read asset** in slsa. `SurgeryData` is the `surgery_data` read asset. This
    standardizes the downstream (slf) interface, because forging reads only the on-disk dataclass. A processor that
    emits a *new* record type rather than `SurgeryData` needs the dataclass, its `ReadAssets` member, and its
    `READ_ASSET_REGISTRY` entry added through the "Adding a new read asset" scenario in `assets:library-extension`.
    Its read and amend surface is added through `assets:data-assets`.
  - A **write processor** (like `WaterLog`) consumes runtime-discovered values and writes them into a
    pre-existing row or record.

The step-by-step procedure covering schema-contract definition, the read/write split, lifecycle implementation,
placement (`cross_system/` versus the system package), wiring, and documentation is the
**"Authoring a custom data-service processor"** workflow in `/acquisition-system-design`. This skill owns the
*what*, meaning the API and the contract, and that workflow owns the *how*, meaning the procedure. For the
seam catalog a new acquisition system composes, see `/library-extension`.

---

## Error model

Processors report failures through `console.error(message=…, error=ValueError)`, which logs and
raises. Construction is atomic, so a processor either constructs cleanly or aborts.

| Symptom                                              | Cause                                                                             |
|------------------------------------------------------|-----------------------------------------------------------------------------------|
| `ValueError` naming missing headers                  | The sheet's header row lacks a required column, from schema drift or a wrong tab. |
| `ValueError`: animal not in the `id` column / no tab | The target animal has no surgery row or no water-log tab.                         |
| `ValueError`: empty header or ID column              | The tab is empty or points at the wrong project or animal.                        |
| `ValueError`: date row not found (`WaterLog`)        | The session's date row is absent, or its date format differs from `M/D/YY`.       |
| `ValueError`: invalid session timestamp              | The `session_date` passed to `WaterLog` is not a valid session name.              |
| `FileNotFoundError` during preprocessing             | A sheet identifier is set but the host credentials file has not been set.         |
| Malformed cell on extract (`float()`/`int()`/date)   | A weight, cage, date, or time cell is empty or non-numeric.                       |

A preprocessing or migration run surfaces these as the failure of the operation that invoked the processor, so
resolve the sheet and re-run. See `/data-management` for the lifecycle-level handling.

### Known pitfalls

- `SurgeryLog._get_column_id` tests `column_name.lower() in self._headers` and then indexes
  `self._headers[column_name]`, so a mixed-case argument raises `KeyError`. Both call sites pass lowercase names,
  `"id"` from `SurgeryLog.__init__` and `"surgery quality"` from `update_surgery_quality`
  (`cross_system/google_sheet_tools.py`).
- `WaterLog._write_value` lowercases its `column_name` before the lookup, so `update_water_log` reaching it with
  `"water given (mL)"` resolves correctly (`cross_system/google_sheet_tools.py`).

---

## Related skills

| Skill                               | Relationship                                                                             |
|-------------------------------------|------------------------------------------------------------------------------------------|
| `/acquisition-system-design`        | Platform-general home of the external data-service processor category and its workflow.  |
| `/data-management`                  | Owns the shared preprocessing primitives that construct these processors.                |
| `/library-extension`                | Catalogues the Sheets processors as a reusable seam a new acquisition system composes.   |
| `/experiment-mcp-environment-setup` | Run first when the `sle mcp` server is not connected.                                    |
| `assets:working-directory`          | Sets and resolves the service-account credentials path the processors require.           |
| `assets:data-assets`                | Reads and amends the on-disk read asset the surgery snapshot produces.                   |
| `assets:library-extension`          | Owns the read-asset registry that receives a new emitted record type.                    |
| `mesoscope:mesoscope-vr`            | Current worked example, which declares the sheet identifiers its own preprocessing uses. |
| `mesoscope:mesoscope-vr-runtime`    | Owns the worked example's session-type branch policy for sheet writes.                   |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines

Before relying on or authoring a Google Sheets processor:
- [ ] Service-account credentials path is set on the host (assets:working-directory)
- [ ] The target sheet is shared with the service-account email (editor access if the processor writes)
- [ ] The sheet identifier(s) are set in the system configuration's external-services section
- [ ] The sheet matches the schema contract (required headers, header-row position, identity model)
- [ ] Gating is honored: no credentials required when all identifiers are unset, required when any is set
- [ ] Every returned processor handle is closed by its owner in a try/finally

When authoring a custom processor:
- [ ] Constructor authenticates, builds the header-to-column map, and validates required headers and record presence
- [ ] Direction is explicit (read into a typed dataclass snapshot, or write into an existing row)
- [ ] A new emitted record type is coordinated via assets:library-extension and assets:data-assets
- [ ] Every API call passes num_retries, and a public close() releases the connection with a __del__ backstop
- [ ] Placed in cross_system/ if system-agnostic, or the system package if system-specific
- [ ] Wired into the system's preprocessing with the unset-identifier skip and credentials-required gating
- [ ] Documented here (or a sibling skill) and in the consuming system's instance skill
```
