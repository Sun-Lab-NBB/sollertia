# Sheet schema contract

The exact structural assumptions and required headers that `SurgeryLog` and `WaterLog` validate at
construction, plus the field-to-header mapping each read produces. Loaded on demand from
`google-sheets-processing`'s SKILL.md. Source: `cross_system/google_sheet_tools.py`.

A processor builds a `header → column-letter` map from the live sheet and asserts every required
header is present before any extract/update call. Header matching is **case-insensitive and
whitespace-stripped** (headers are lowercased on read). A column whose required header is missing
aborts construction with a `ValueError` naming the missing headers.

---

## Cross-cutting conventions

| Convention               | Rule                                                                                                         |
|--------------------------|--------------------------------------------------------------------------------------------------------------|
| Header matching          | Lowercased and stripped, so the live header text must match the required header after that normalization.    |
| Animal ID format         | Zero-padded to a five-digit string for comparison (`12` → `00012`).                                          |
| Empty-value placeholders | `""`, `n/a`, `--`, `---` (case-insensitive) are read as `None`.                                              |
| Column-letter mapping    | 0-based column index → Excel-style letter (`A`, `B`, … `Z`, `AA`, …).                                        |
| Surgery-log date formats | `%m-%d-%y`, `%m-%d-%Y`, `%m/%d/%y`, `%m/%d/%Y`. Times are `%H:%M`.                                           |
| Timezone                 | Human-entered dates/times are interpreted as host **local time** and stored as UTC microseconds-since-epoch. |
| Write formatting         | Written cells get CENTER horizontal / MIDDLE vertical alignment via a follow-up `batchUpdate`.               |
| Retries                  | Every API call uses `num_retries=5` (`_GOOGLE_API_MAX_RETRIES`) for transient 5xx/429 errors.                |

The `_SUPPORTED_DATE_FORMATS` set feeds `_convert_date_time_to_timestamp`, which parses the surgery log's date and
time cells. The water log resolves its own `date` column by string comparison, under the format given in
[Water restriction log](#water-restriction-log-waterlog).

---

## Surgery log (`SurgeryLog`)

| Structural assumption | Value                                                                                                                                                  |
|-----------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Tab identity          | One tab per **project**, with the tab name set to `project_name`.                                                                                      |
| Header row            | Row **1**.                                                                                                                                             |
| Data rows             | Row 2 onward.                                                                                                                                          |
| Record identity       | The **`id` column** (zero-padded five-digit animal IDs). The target animal's row index is its position in that column + 2 (header row + 0→1 indexing). |

### Required headers

The `_REQUIRED_SURGERY_HEADERS` set, grouped by the dataclass section each feeds:

- **Subject:** `id`, `ear punch`, `sex`, `genotype`, `dob`, `weight (g)`, `cage #`, `location housed`, `status`
- **Procedure:** `date`, `start`, `end`, `surgeon`, `protocol`, `surgery notes`, `post-op notes`, `surgery quality`
- **Drug:** `lrs (ml)`, `ketoprofen (ml)`, `buprenorphine (ml)`, `dexamethasone (ml)`

### Dynamic implant / injection columns

Implants and injections are discovered by header name, not fixed columns. Headers follow the
`implant<N>` / `injection<N>` convention, with companion columns sharing the same prefix:

| Header pattern             | Maps to                                            |
|----------------------------|----------------------------------------------------|
| `implant<N>`               | `ImplantData.implant` (the implant name)           |
| `implant<N> location`      | `ImplantData.implant_target`                       |
| `implant<N> coordinates`   | Parsed into AP/ML/DV (optional)                    |
| `implant<N> code`          | `ImplantData.implant_code` (defaults to `"0"`)     |
| `injection<N>`             | `InjectionData.injection`                          |
| `injection<N> location`    | `InjectionData.injection_target`                   |
| `injection<N> volume (nl)` | `InjectionData.injection_volume_nl`                |
| `injection<N> coordinates` | Parsed into AP/ML/DV (optional)                    |
| `injection<N> code`        | `InjectionData.injection_code` (defaults to `"0"`) |

The "main" column (no spaces, e.g. `implant1`) drives detection. A `None` value in that cell means
the animal does not have that implant or injection even though the header exists. Drug, implant, and
injection `code` columns are optional and default to the string `"0"`, which reads as "no code", for
backward compatibility with early sheet versions.

Once a main column holds a name, the reader indexes `<base> location` and `injection<N> volume (nl)`
directly, so a missing companion header raises `KeyError`. An empty `injection<N> volume (nl)` cell
additionally raises `TypeError` from the `float()` conversion, while an empty `<base> location` cell
is stored as `None` without error. The `coordinates` and `code` companions are read through `.get()`
and stay optional in the implant and injection loops of `SurgeryLog.extract_animal_data`
(`cross_system/google_sheet_tools.py`).

### Coordinate string format

Stereotactic coordinates are a single string like `-1.8 AP, 2 ML, .25 DV`, parsed into an
`(AP, ML, DV)` float tuple. A surgery without coordinates (e.g., training) defaults to `(0, 0, 0)`.

### Output: `SurgeryData`

`extract_animal_data()` returns `SurgeryData(subject, procedure, drugs[], implants[], injections[])`.
Notable per-field parsing: `dob` is combined with a noon time, `date` plus `start`/`end` become
`surgery_start_us`/`surgery_end_us`, `weight (g)` becomes a `float`, and `cage #` becomes an `int`.
A blank `surgery quality` cell resolves to `0`. An empty `weight (g)`, `cage #`, or `id` cell is read as `None` and
raises `TypeError` from the `float()` or `int()` conversion, while a malformed `id`, `weight (g)`, `cage #`, or
`surgery quality` cell raises `ValueError` from the same conversion. An empty or malformed `date`, `start`, or `end`
cell raises `ValueError` from `_convert_date_time_to_timestamp`. All of these surface from the argument expressions
evaluated inside `SurgeryLog.extract_animal_data`, not from the `SubjectData` and `ProcedureData` dataclasses, which
perform no validation (`cross_system/google_sheet_tools.py`).

Each drug tracked by `_SURGERY_LOG_DRUGS` becomes a named `DrugData` record in `drugs[]`, covering
`Lactated Ringer's Solution`/`lrs`, `Ketoprofen`/`ketoprofen`, `Buprenorphine`/`buprenorphine`, and
`Dexamethasone`/`dexamethasone`. For each record, `drug` is the descriptive name, `drug_volume_ml` comes
from the `<stem> (ml)` column, and `drug_code` comes from the `<stem> code` column. A drug whose volume
cell is empty was not administered and is excluded from `drugs[]`.

---

## Water restriction log (`WaterLog`)

| Structural assumption | Value                                                                                                   |
|-----------------------|---------------------------------------------------------------------------------------------------------|
| Tab identity          | One tab per **animal**, with the tab name set to the numeric animal ID (digit-only tabs only).          |
| Header row            | Row **2** (differs from the surgery log's row 1).                                                       |
| Data rows             | Row 3 onward.                                                                                           |
| Record identity       | The session's date must already exist in the pre-filled **`date` column**. That row is the session row. |
| Date match format     | Non-zero-padded `M/D/YY` (`%-m/%-d/%y`, e.g. `5/24/26`), matched by exact string equality.              |

### Required headers

The `_REQUIRED_WATER_RESTRICTION_HEADERS` set: `date`, `weight (g)`, `given by:`, `water given (ml)`,
`behavior`, `time`.

### Written cells

`update_water_log(weight, water_ml, experimenter_id, session_type)` writes the session row:

| Column             | Value                                                               |
|--------------------|---------------------------------------------------------------------|
| `weight (g)`       | Animal weight at session onset (rounded to 1 decimal).              |
| `given by:`        | `experimenter_id`.                                                  |
| `water given (ml)` | Combined runtime + post-runtime water in mL (rounded to 1 decimal). |
| `behavior`         | `session_type`.                                                     |
| `time`             | Session start time in `HH:MM` local time (cached at construction).  |

The log must be **pre-filled with session dates**, because `WaterLog` writes into an existing date row and creates none.
An absent session date raises `ValueError` naming the date and telling the operator to update the log and rerun the
failed command. The lookup compares each `date` cell against `local_datetime.strftime("%-m/%-d/%y")` for exact equality,
so a cell holding `05/24/26` or `05-24-26` fails to match the session's `5/24/26` and aborts construction. The surgery
log tries four `strptime` patterns, and the water log matches one exact string.

---

## What a custom processor must redefine

When adapting to a different schema or service, the parts above are exactly the parts that change.
A custom processor declares its own:

1. **Required-header (or required-field) set**, the equivalent of `_REQUIRED_*_HEADERS`, validated
   at construction.
2. **Structural assumptions**, meaning which row holds headers, whether record identity is a column
   value or a tab name, and where data rows begin.
3. **Field mapping**, meaning how source columns map to the typed record it emits (read) or which
   columns it writes (write).

Everything in [Cross-cutting conventions](#cross-cutting-conventions) is reusable as-is, covering auth,
retries, connection teardown, and write formatting. Preserve it so the processor behaves like the rest
of the platform.
