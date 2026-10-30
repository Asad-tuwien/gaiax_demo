# AAS / DPP Metadata Compliance Checker

Checks an AAS JSON environment that contains a `DppMetadata` submodel as defined in
**IDTA 02099-1 v1.0 (28.05.2026)**. Available as a command-line tool and as a small
local web UI. Python 3.8+, standard library only (optional: `pip install jsonschema`
for `--aas-schema`).

> **Scope:** the tool is *compatible with* the rules listed below. It is **not** a
> certification of compliance with any standard.

---

## What it does

You give it an **Asset Administration Shell (AAS)** file in JSON format that carries a
**Digital Product Passport (DPP)** metadata submodel. It tells you, check by check,
whether the file is valid JSON, has a sound AAS structure, and whether the DPP metadata
follows IDTA 02099-1: the right attributes, the right types, valid values and a valid
timestamp. Every result is `PASS`, `FAIL`, `WARN` or `SKIP`, with the location, the
expected and actual value, and the source the rule comes from. Terms such as *submodel*,
`idShort` and `semanticId` are explained in the [glossary](#12-glossary).

## Quick start

```
cd dpp_compliance_checker
python demo.py                       # generates fixtures/, runs 4 scenarios, prints reports
python dpp_checker.py fixtures/valid_dpp.json
python server.py                     # web UI at http://127.0.0.1:8000
```
On Windows use `python` or `py`; on Linux/macOS `python3`. Python 3.8+ is required.
`python demo.py` ends with `Demo outcome as expected.` if everything works.

**Sample output** (`fixtures/valid_dpp.json`, shortened; a passing check is one line):
```
[Layer 3 IDTA 02099-1]  -> PASS
  [PASS] MAND-lastUpdate | MANDATORY | Mandatory attribute 'lastUpdate' present
  [PASS] TYPE-valueType-lastUpdate | TYPE | Declared valueType
  ...
Totals: PASS=...  FAIL=0  WARN=0  SKIP=1
OVERALL RESULT: COMPLIANT
(with respect to the implemented checks only - this is not a certification; SKIPped checks were not evaluated)
```
**A failing file** (`fixtures/invalid_dpp.json`, only FAIL lines shown):
```
  [FAIL] MAND-dppStatus | MANDATORY | Mandatory attribute 'dppStatus' present
  [FAIL] MAND-economicOperatorId | MANDATORY | Mandatory attribute 'economicOperatorId' present
  [FAIL] TYPE-modelType-granularity | STRUCTURAL | Element type (modelType)
  [FAIL] TYPE-value-facilityId | TYPE | Actual value representation
  [FAIL] TYPE-listItems-contentSpecificationIds | TYPE | SML items are Property/xs:string ...
  [FAIL] DT-lastUpdate-wellformed | DATETIME | Well-formed date-time (syntax and calendar)
Failures by category: MANDATORY=2, STRUCTURAL=1, TYPE=2, DATETIME=1, SEMANTIC=1
OVERALL RESULT: NOT COMPLIANT
```
**One failing check in full** (visible for every FAIL/WARN; add `-v` to also show PASS details and sources):
```
  [FAIL] DT-lastUpdate-profile | DATETIME | Profile: timezone requirement
         location: submodelElements[idShort=lastUpdate].value
         result:   Valid timestamp but violates the profile: timezone designator (Z or +hh:mm) is required but absent.
         expected: timezone-aware
         actual:   "2026-10-21T15:53:00"
```
The exact number of checks changes with the file, because some checks only exist for
attributes that are present.

---

## 1. Project layout

```
dpp_compliance_checker/
  dpp_checker.py        checker (library + CLI)
  server.py             local web server for the UI (uses dpp_checker.py unchanged)
  demo.py               generates fixtures, runs four demo scenarios
  fixtures/             valid_dpp.json, invalid_dpp.json, no_timezone_dpp.json, malformed.json
  frontend/
    index.html  style.css  app.js
    img/                your logos and milling-cutter picture (logo1.png, logo2.png, milling-cutter.png)
```

## 2. Running

**Web UI**
```
python server.py
```
Open http://127.0.0.1:8000, then drop or select an AAS JSON file. The server listens on
localhost only, accepts files up to 10 MB, and keeps nothing (the upload is a temporary
file that is deleted after the check). `index.html` must be in `frontend/` (the server
exits with a message if it cannot find it).

**Command line**
```
python dpp_checker.py fixtures/valid_dpp.json
python dpp_checker.py my_file.json -v              # details + source for every check
python dpp_checker.py my_file.json --json          # machine-readable report
python demo.py                                     # four scenarios: pass, fail, no timezone, malformed
```
Exit code: `0` compliant, `1` not compliant.

| Option | Effect |
|---|---|
| `--semantic-ids strict\|warn\|off` | semanticId mismatch = FAIL (default) / WARN / ignored |
| `--require-utc` | `lastUpdate` must end in `Z` or `+00:00` |
| `--allow-no-timezone` | do not require a timezone on `lastUpdate` |
| `--strict-content-spec` | empty `contentSpecificationIds` list = FAIL |
| `--submodel-id ID` | explicit `id` of the DPP submodel (use when several candidates exist) |
| `--aas-schema FILE` | also validate against a versioned AAS JSON schema (needs `jsonschema`) |

## 3. Validation layers

Each layer is reported separately; the overall result needs all of them to pass.

1. **JSON** – file readable, UTF-8, syntactically valid JSON, root is an object.
2. **AAS structure** – *lightweight*: required root collections, each AAS has `id` and
   `assetInformation`, submodel references are well-formed and resolve, every submodel
   has an `id`, the DPP submodel can be resolved. **This is not full AAS schema
   validation** unless a schema is supplied with `--aas-schema`.
3. **IDTA 02099-1** – the DPP metadata submodel itself (details below).

The overall verdict is `COMPLIANT` only if no check is `FAIL` and validation ran to
completion. `WARN` and `SKIP` do not change the verdict but are always shown.

## 4. Checks in detail

**Locating the submodel.** The submodel is recognised by `idShort = DppMetadata` or
semanticId `https://admin-shell.io/idta/cds/dppMetadata/1`, narrowed to those referenced
by an AAS. Other submodels in the file are ignored for the DPP checks (they are only
checked for having an `id` and for resolvable references). If more than one candidate
remains, the checker refuses to guess and fails; set `--submodel-id`. Elements are found
by `idShort`, never by array position; duplicates are reported.

**Mandatory vs optional.** Whether an attribute is mandatory is decided by the
**`SMT/Cardinality` qualifier in the JSON** (`One`, `OneToMany` = mandatory;
`ZeroToOne`, `ZeroToMany` = optional). If an element has no usable qualifier or is
absent, the IDTA 02099-1 default applies (7 mandatory attributes: `digitalProductPassportId`,
`uniqueProductIdentifier`, `granularity`, `dppSchemaVersion`, `dppStatus`, `lastUpdate`,
`economicOperatorId`; optional: `facilityId`, `contentSpecificationIds`). A consistency
check (`CARD-*`) compares the qualifier with the specification table and warns on
disagreement. *Trade-off:* a file can weaken a requirement by changing its qualifier; to
make that a failure instead of a warning, edit `check_cardinality_consistency`.
A mandatory attribute passes only if its element exists and its value is non-empty
(whitespace-only strings fail). Missing attributes are reported individually.
Example-value qualifiers are **never** used as values.

**Element and value types.** Per attribute: `modelType` (structural error if wrong),
declared `valueType` (`xs:string`, `xs:dateTime`) and the actual JSON type (scalar values
must be JSON strings). `contentSpecificationIds` must be a `SubmodelElementList` with
`typeValueListElement = Property`, `valueTypeListElement = xs:string`, and items that are
`Property` objects with non-empty string values (values are read from the nested objects,
not assumed to be plain strings).

**Optional attributes.** Absence is a PASS. If present, the same structure and type rules
apply. Present-but-blank scalar = WARN.

**semanticIds and allowed values.** Element semanticIds are compared with the IDTA table
(`https://admin-shell.io/idta/cds/<name>/1`; list items: `.../contentSpecificationId/1`).
`granularity` must be one of `Model`, `Item`, `Batch` (case-sensitive). No enumeration is
enforced for `dppStatus` (none is defined) unless you configure one.

**Date and time (`lastUpdate`).** Four stages, using explicit logic, not a regex alone
(a pattern only splits the string into parts):
1. Value is a JSON string.
2. Well-formed `YYYY-MM-DDThh:mm:ss[.fraction][Z|±hh:mm]`: month 1–12, day valid for that
   month and year (leap years handled), hour 0–23, minute 0–59, second 0–59, year ≥ 1.
3. Timezone designator valid: offset within −14:00…+14:00, minutes ≤ 59.
4. Profile: timezone present (IDTA says "UTC-based"); with `--require-utc`, only `Z`/`+00:00`.

A syntactically invalid timestamp (stage 2/3) is reported separately from a valid one that
violates the profile (stage 4). Examples: `2026-10-21T15:53:00Z` accepted;
`2026-10-21T15:53:00` valid but fails the profile; `2026-02-30T10:00:00Z`,
`2026-13-01T10:00:00Z`, `2026-10-21T25:00:00Z`, `2026-10-21T15:53Z` (no seconds),
`2026-10-21T24:00:00Z`, `…+15:00` rejected. Not supported: negative or 5-digit years,
`24:00:00`, leap seconds (stricter than XML Schema `xs:dateTime`).
**Plausibility (project rule, not in the specification):** a timestamp more than
`future_tolerance_minutes` (default 5) in the future, or earlier than
`earliest_plausible_year` (default 2000), gives a **WARN** (`DT-lastUpdate-plausible`).
Set either to `None` in `CheckerConfig` to disable, or change the WARN to FAIL to enforce.

## 5. Web UI

- Start screen: "Drop or select a file here".
- Results appear as steps in order of importance, each with a status icon and a dropdown
  showing every check, where it looked, expected/actual values and the source it was
  checked against: **Valid JSON file → AAS structure → Mandatory fields → Attributes &
  types (key/value table) → Date, time & timezone → Semantic IDs & allowed values**.
- Summary with overall verdict and one chip per layer (JSON, AAS structure, IDTA 02099-1).
- Logos: `frontend/img/`, referenced in `index.html`; the picture next to the verdict is
  chosen in `toolImageFor()` in `app.js` (currently always `milling-cutter.png`).
- Colours: `:root` in `style.css`. Layout width: `main{max-width…}` in `style.css`.
- The key/value table is read from the uploaded file in the browser for display only; all
  pass/fail decisions come from the Python checker. New checks added in Python appear in
  the UI automatically if their `check_id` prefix is mapped in `stepOf()` (`app.js`).

## 6. Sources and what is (not) verified

**Verified** against the published IDTA 02099-1 v1.0 PDF (28.05.2026), sections 3.1/3.2:
submodel idShort and semanticId; attribute table (element types, value types, cardinality,
semanticIds); `granularity` values `Model`/`Item`/`Batch`; `lastUpdate` as `xs:dateTime`,
"UTC-based according to ISO 8601"; `contentSpecificationIds` list structure and item
semanticId; no `dppStatus` enumeration and no schema-version format are defined.

**Not verified / not used:**
- **EN 18223:2026 and EN 18219:2026** – full texts were not available. They are referenced
  only where IDTA 02099-1 quotes them. **No identifier-format rules** (product, operator or
  facility identifiers) are checked; that layer was removed.
- **AAS metamodel (IDTA-01001)** – the structural checks reflect the general AAS v3 JSON
  structure; exact version and clause numbers were not verified.

Every result carries a `source` (shown with `-v`, in `--json` output and in the UI details).

## 7. Interpretation decisions (configurable in `CheckerConfig`)

- Empty `contentSpecificationIds` list: allowed by the project requirements (WARN); IDTA
  02099-1 §3.2 gives the item cardinality as `[1..*]`. `--strict-content-spec` makes it FAIL.
- Non-zero timezone offsets are accepted unless `--require-utc` (the spec's "UTC-based"
  is not unambiguous on this).
- semanticId mismatch is FAIL by default (`--semantic-ids warn|off` to relax).
- Plausibility of `lastUpdate` is a WARN, not a specification requirement.

## 8. Not implemented

Full AAS schema validation (only via `--aas-schema`), concept-description content checks,
EN 18223 payload rules, EN 18219 identifier formats, content-specification ID resolution,
data carrier / access-control requirements, checking elements other than the nine IDTA
02099-1 attributes.

## 9. Extending

- New check: write `check_xyz(index, cfg) -> List[CheckResult]` and call it in
  `run_compliance_checks`. Give it a `check_id` prefix and map it in `stepOf()` (`app.js`).
- New attribute: add an `AttributeRule` to `ATTRIBUTE_RULES`.
- New setting: add a field to `CheckerConfig`; expose it via `main()` (CLI) or `server.py`.

## 10. How it works

### Data flow
```
 Browser (frontend/)                         Python
 ───────────────────                         ──────
 app.js  --POST raw file-->  server.py  --temp file-->  run_compliance_checks(path, CheckerConfig)
   ▲                            │                         ├─ load_json_file              (layer 1)
   │                            │                         ├─ validate_root_structure     (layer 1/2)
   │   JSON report              │                         ├─ validate_aas_structure      (layer 2)
   └────────────────────────────┘ <-- report_to_dict ---  ├─ find_dpp_submodel           (layer 2)
 groups results into steps                                └─ mandatory / types / datetime /
 by check_id prefix (stepOf)                                 optional / semantic / values  (layer 3)
```
The command line uses the same `run_compliance_checks()` and prints with `print_report()`.
Validation logic never prints and never touches the network; I/O lives in `print_report`,
`main` and `server.py`.

### Result format
Every check returns a `CheckResult`; the UI and the `--json` output are built from these.

| Field | Meaning |
|---|---|
| `check_id` | Stable id, e.g. `MAND-lastUpdate`, `DT-lastUpdate-wellformed` |
| `name` | Human-readable title |
| `layer` | `1 JSON`, `2 AAS structure` or `3 IDTA 02099-1` |
| `category` | `INPUT_FILE`, `MANDATORY`, `STRUCTURAL`, `TYPE`, `DATETIME`, `SEMANTIC`, `VALUE`, `OPTIONAL`, `INTERNAL` |
| `location` | Where in the file the check looked |
| `status` | `PASS`, `FAIL`, `WARN` (reported, does not fail), `SKIP` (not evaluated) |
| `message`, `expected`, `actual` | Explanation and, where useful, the expected and actual values (long values are truncated) |
| `source` | Specification the rule comes from, or "project requirement" |

`check_id` prefixes are used by the UI to group results into steps (`stepOf()` in `app.js`):
`JSON-`/`ROOT-OBJECT` file, `ROOT-`/`AAS-`/`DPP-SUBMODEL` structure, `MAND-` mandatory,
`TYPE-`/`ELEM-`/`OPT-`/`CARD-`/`DPP-IDSHORT` attributes, `DT-` date and time,
`SEM-`/`VAL-`/`DPP-SEMANTIC` semantic IDs and values.

The report also contains `verdict`, `validation_complete` and `incomplete_reason`.
If an early step fails so badly that later checks cannot run safely (unparseable file,
missing `submodels` collection, DPP submodel not found), the later layers are listed as
`SKIP` and the verdict is `NOT COMPLIANT` with `validation_complete = false`. A broken file
can never come out as compliant.

### Design principles
- **No guessing.** If several submodels could be the DPP one, the check fails instead of
  picking one. Elements are found by `idShort`, never by array position.
- **No crashes on bad input.** Missing keys, `null`, wrong JSON types, duplicates and
  malformed nested objects are all handled; unexpected exceptions become an `INTERNAL`
  failure instead of a stack trace.
- **Independent checks keep going.** One failing attribute does not stop the others.
- **Sources over assumptions.** Rules come from the verified specification or are marked
  as project rules; interpretation choices live in `CheckerConfig`.
- **Metadata is not data.** Descriptions, concept descriptions and example-value
  qualifiers are never treated as the attribute's value.

## 11. Troubleshooting

| Problem | Cause and fix |
|---|---|
| Browser shows `404 File not found` | `index.html` is not in `frontend/` next to `server.py`. Move the three front-end files there. |
| `ModuleNotFoundError: dpp_checker` | Run `server.py` / `demo.py` from the folder that contains `dpp_checker.py`. |
| `Could not reach the checker. Is server.py running?` | Start `python server.py` and open the page via `http://127.0.0.1:8000`, not as a local file. |
| Port 8000 already in use | Change `PORT` in `server.py`. |
| Changes to CSS/JS do not show | Hard refresh (Ctrl+F5). |
| `KeyboardInterrupt` traceback in the terminal | Normal when stopping the server with Ctrl+C. |
| Logos or picture missing | Check the file names in `frontend/img/` and the `src` paths. A missing milling-cutter image removes itself silently. |
| A valid template file is "not compliant" | A *template* has example values only in qualifiers, not in `value`. The checker validates DPP *instances*, so mandatory values are missing. |
| Everything fails with "DPP metadata submodel not found" | The submodel needs `idShort` `DppMetadata` or semanticId `https://admin-shell.io/idta/cds/dppMetadata/1`. |
| "candidate DPP submodels found" | Several matches. Pass `--submodel-id <id>` or set `CheckerConfig.submodel_id`. |
| semanticId failures on your own data | Your data uses other semantic IDs. Use `--semantic-ids warn` or `off`. |
| `--aas-schema` fails | Install `jsonschema` (`pip install jsonschema`) and pass a valid schema file. |

## 12. Glossary

- **AAS (Asset Administration Shell):** standardised digital representation of an asset
  (here: a milling cutter). Serialised as JSON with the root collections
  `assetAdministrationShells`, `submodels` and `conceptDescriptions`.
- **Submodel:** a themed block of data inside the AAS; here the `DppMetadata` submodel.
- **Submodel element:** one data item in a submodel, e.g. a `Property` (single value) or a
  `SubmodelElementList` (list of values). The element kind is its `modelType`.
- **`idShort`:** the short local name of an element (e.g. `lastUpdate`).
- **`semanticId`:** a reference that says what an element means; compared with the IDs in the specification.
- **Qualifier:** extra metadata on an element. `SMT/Cardinality` says whether it is required; `SMT/ExampleValue` is only an example and is ignored.
- **Concept description:** semantic definition of a concept. It describes meaning; it is not the attribute's value.
- **`valueType` (`xs:string`, `xs:dateTime`):** the declared data type of a Property.
- **DPP (Digital Product Passport):** digital record of a product's characteristics through its life cycle.
- **IDTA 02099-1:** the specification of the DPP metadata submodel this tool checks against.
- **Template vs instance:** a template defines the structure with example values; an instance holds real values. This tool checks instances.