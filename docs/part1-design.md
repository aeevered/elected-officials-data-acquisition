# Part 1: County Elected Officials — Design & Planning

**Purpose:** Propose a maintainable way to represent U.S. county elected officials, choose sources responsibly, and run a system that keeps the dataset reasonably current despite structural messiness at the county level.

---

## 1. Data model

### Design goals

- **Represent reality, not a simplified org chart.** The same title can mean different things in different states; some counties elect roles others appoint; some offices are county-wide, others are district-based.
- **Separate “what the office is” from “who holds it.”** That makes updates, history, and corrections easier.
- **Keep provenance.** Every fact worth trusting should trace back to a source and a retrieval time.
- **Support partial coverage.** It should be normal for a county–office pair to be unknown or stale, without breaking the schema.

### Core entities (conceptual)

Think relational database or equivalent parquet/JSON with stable IDs; the ideas matter more than a specific engine.

**1. `jurisdiction`**

- Represents a county (or county-equivalent) as the scope of local government.
- Fields (illustrative): `jurisdiction_id`, `state_fips`, `county_fips` (or GNIS / Census GEOID), `name`, `name_aliases[]`, `classification` (e.g., parish, borough), `parent_state_code`.
- *Why:* Analysts need to join to Census, elections, and other datasets; FIPS/GEOID are the usual keys even when names collide (“Jefferson County” in many states).

**2. `office` (or split into `office_type` + `office_instance`)**

- **`office_type`:** A normalized concept of the role — e.g., “Sheriff,” “County Clerk,” “County Commissioner,” “County Executive” — with optional national taxonomy codes if we adopt one later (e.g., from a standards body or internal enum).
- **`office_instance`:** The actual office *in* a jurisdiction: “Sheriff of Wake County, NC” or “Commissioner District 3, Harris County, TX.”
- Fields for `office_instance`: `office_instance_id`, `jurisdiction_id`, `office_type_id`, `district_id` (nullable), `seat_number` (nullable), `is_elected` (boolean, nullable if unknown), `term_length_years` (nullable), `notes` (free text for oddball cases).

*Variability:* Not every county has every `office_type`. That is modeled by **absence of rows** in `office_instance`, not NULL columns on a wide “county spreadsheet.” Optional supplements:

- **`jurisdiction_office_catalog`:** For a given state or county, which offices *exist* structurally (from a constitution, charter, or state statute summary), even if we do not yet have an officeholder. Helps distinguish “we have not collected” from “this office does not exist here.”

**3. `person`**

- `person_id`, `full_name`, `name_components` (parsed given/middle/family/suffix), `suffix`, `known_aliases[]`.
- Avoid over-identifying people in the public dataset if not required (e.g., DOB is usually unnecessary for this use case).

**4. `person_office_appointment` (or `tenure`)**

- Links a person to an `office_instance` for a period.
- Fields: `appointment_id`, `person_id`, `office_instance_id`, `role_title_raw` (string as seen on source), `start_date`, `end_date` (nullable for incumbents), `is_acting`, `election_id` (optional FK if we ingest elections later), `source_record_ids[]` / lineage (see below).

*Why a tenure table:* Handles special elections, resignations, interim appointments, and mid-term party switches without rewriting person rows.

**5. `source` and `source_record`**

- **`source`:** Catalog of origins — “Wake County Board of Commissioners roster page,” “Texas Secretary of State — county officials,” etc. Fields: `source_id`, `source_type` (government_site, state_agency, third_party_aggregate, manual_research), `base_url`, `trust_tier` (see §2), `update_frequency_hint`.
- **`source_record`:** A blob or structured extract from one fetch: `source_record_id`, `source_id`, `fetched_at`, `content_hash`, `raw_payload_ref` (S3 path, blob storage), `parser_version`, `extracted_fields` (JSON). Downstream normalized rows reference `source_record_id` so we can re-parse when parsers improve.

**6. `data_quality_flag` (optional but valuable)**

- Attach to `office_instance`, `person_office_appointment`, or fields: `flag_type` (stale, conflicting_sources, inferred_district, name_parse_uncertain), `severity`, `notes`, `created_at`.

### Flat-file alternative

If the team prefers files over a database initially:

- **`counties.csv`:** jurisdiction attributes.
- **`offices.csv`:** office_instance rows.
- **`people.csv`:** person rows.
- **`tenures.csv`:** appointment rows with `source_record_id`.
- **`sources.csv`** + **`source_snapshots/`** (one file per fetch, keyed by hash).

The same normalization rules apply; you trade query convenience for pipeline simplicity.

### Handling state/county variability (summary)

| Challenge | Modeling approach |
|-----------|-------------------|
| Different office sets per county | Sparse `office_instance` rows; optional `jurisdiction_office_catalog` for structural expectations |
| Same title, different powers | `office_type` + state-specific metadata table or `notes`; avoid one national row per “Clerk” without context |
| Districted vs at-large | `district_id` on `office_instance`; nullable when at-large |
| Elected vs appointed | `is_elected` on instance or tenure; allow NULL when unclear |
| Name changes / consolidations | `jurisdiction` aliases + historical jurisdiction mapping table if needed later |

---

## 2. Source strategy

### Candidate sources (illustrative, not exhaustive)

| Tier | Examples | Strengths | Weaknesses |
|------|----------|-----------|------------|
| **A — Primary government** | County official websites, county clerks, sheriffs’ sites, state associations of counties | Highest legitimacy for “who holds office now” | Fragmented formats; sites go down; PDFs; no API |
| **A — State constitutional officers lists** | Some SOS or state comptroller “local government” directories | Sometimes structured; covers many counties | Varying freshness; may omit district detail |
| **B — Curated third parties** | NACo directories, Ballotpedia, OpenStates (where applicable), vendors | Faster coverage; often cleaned | License/terms; lag vs ground truth; aggregation errors |
| **C — Secondary** | News, press releases, meeting minutes | Fills gaps when sites are stale | Not authoritative; harder to automate |

### What makes a source “trustworthy” here

- **Authority:** Produced or clearly endorsed by the government body that administers the office or election.
- **Transparency:** Dated pages, contact info, and (ideally) a clear “last updated” signal.
- **Stability:** URL and DOM patterns stable enough to automate, or a downloadable file with a predictable schedule.
- **Verifiability:** A second independent source (e.g., county + state) can corroborate high-stakes rows.

I would treat **tier A** as default truth when fresh; use **tier B** for bootstrapping and gap-fill with explicit `trust_tier` and flags; use **tier C** only with human review or as a trigger to re-fetch tier A.

### Expected gaps and mitigations

| Gap | Mitigation |
|-----|------------|
| No county website | Fall back to state directory; flag low confidence; queue for manual or crowdsourced verification |
| PDF rosters only | OCR + layout parsers; higher error rate → QA sampling |
| District boundaries unclear | Store district label as `raw`; join to spatial data later in a separate initiative |
| Rapid turnover (special elections) | Shorter refresh cadence in high-change windows; monitor news feeds for trigger re-crawl |
| Conflicting sources | Retain both `source_record`s; surface conflict in `data_quality_flag`; rule: prefer newer tier A |

---

## 3. Collection approach (high-level architecture)

### Pipeline stages

1. **Inventory** — Maintain a registry of counties and, per county, candidate URLs and source priorities (from a mix of Census place lists, state indexes, and manual seeds).
2. **Schedule** — Orchestrator (cron, queue workers, or managed scheduler) assigns work by `source` and `jurisdiction`, with rate limits and robots.txt respect.
3. **Fetch** — HTTP downloads, browser automation only when necessary (slower, flakier). Store immutable `source_record` blobs with timestamps.
4. **Extract** — Source-specific parsers (HTML tables, PDFs, JSON if any). Version parsers; re-run on historical blobs when parsers improve.
5. **Normalize** — Map raw strings to `office_type`, `jurisdiction`, `person` (fuzzy name match with caution), `district`. Emit proposed `tenure` rows.
6. **Validate** — Rule engine: required fields, date sanity, duplicate detection, “incumbent count” per seat should be ≤ 1, etc.
7. **Publish** — Materialize analyst-facing tables or files; expose `last_updated` per jurisdiction and per office_instance.
8. **Monitor** — Dashboard: coverage %, stale sources, parse failure rates, open flags.

### Keeping data current

- **Tiered cadence:** High-population or high-churn counties more often; long-tail counties on a slower cycle with random jitter to avoid thundering herds.
- **Change detection:** Store hashes of normalized extracts; skip deep reprocessing when nothing changed; still re-fetch periodically to detect silent page changes.
- **Event-driven refresh:** Optional hooks from election calendars (where we have them) to bump priority after known election dates.

### Non-goals for an initial build

- Perfect real-time accuracy everywhere.
- Fully automated resolution of every ambiguity without human review.

A practical v1 accepts **coverage and freshness metrics** as first-class outputs so stakeholders know what they are getting.

---

## 4. Tradeoffs & open questions

### Tradeoffs (explicit)

- **Rich relational model vs flat wide tables:** Relational is harder for casual spreadsheet users but essential for history, provenance, and varying office sets. Mitigation: publish flattened “reporting” views.
- **Automation vs quality:** Aggressive scraping scales; edge cases need human time. I would automate the happy path and queue exceptions.
- **National normalization vs state-first:** Starting **state-by-state** (normalize within state, then map to national `office_type`) often reduces errors than forcing one national ontology on day one.

### Assumptions

- County and county-equivalent boundaries can be aligned to a standard geographic identifier (e.g., Census) for most analytic use cases.
- “Elected official” can be operationalized with `is_elected = true` where sources support it; appointed roles may be out of scope or tracked separately depending on product needs.

### Known weaknesses

- **Incumbent detection without an election database** risks missing turnovers between crawls.
- **Name disambiguation** (two “J. Smith”s) cannot be solved from rosters alone; may need extra attributes or manual keys.
- **Parser maintenance** is ongoing; site redesigns break scrapers — budget for it.

### With more time

- User research with analysts on must-have dimensions (party, contact info, photos).
- Pilot **2–3 states** end-to-end to calibrate effort per county before national rollout.
- Formal **data contract** for downstream systems (SLA on freshness, schema versioning).

---

## AI usage notes (Part 1)

*Edit this subsection before submission so it accurately reflects your process.*

- **AI assistance:** *[e.g., None / Used for outlining, structure, and example field lists; revised all sections for accuracy and personal judgment.]*
- **Verification:** *[e.g., Cross-checked FIPS/terminology against Census practice; adjusted source tiers to match how I would actually prioritize work; removed suggestions that oversimplified district-level cases.]*
- **Own work:** *[e.g., Final tradeoff choices, validation rules I would enforce first, and state-first vs national ontology stance.]*

*(Add a separate short note for Part 2 when you complete it.)*
