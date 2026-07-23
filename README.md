# OSIRIS RWD — Common Data Model for Real-World Oncology Data

**Version 2.0.0** · June 2026 · An [INCa](https://www.e-cancer.fr/) (Institut National du Cancer) project, coordinated with the support of Unicancer

OSIRIS RWD is a common data model for **Real-World Data** in oncology. It defines a minimum, standardized, event-based schema for sharing clinical oncology data extracted from hospital information systems (HIS / *SIH*), with the explicit goal of enabling **automatic, unambiguous population** of the database rather than manual data entry.

The model was specified by a multi-institution Working Group (comprehensive cancer centres, university hospitals, and partner organizations). It is freely released to the community and is expected to evolve with usage and field feedback.

---

## Why OSIRIS RWD

OSIRIS RWD targets **routine-care, real-world conditions**, where data quality, completeness, and provenance differ substantially from curated trial data. The design rests on a few explicit principles:

- **Automatic data population first.** Every variable has a precisely defined population rule so it can be populated programmatically from existing HIS sources without ambiguity.
- **Structured sources over unstructured ones.** Coded administrative/clinical data (e.g. PMSI) is always preferred to free text (e.g. reports) when both are available.
- **International terminologies** are used wherever they exist.
- **Referentials are chosen pragmatically** — by their actual use in HIS, the availability of crosswalks, ANS (*Agence du Numérique en Santé*) recommendations, granularity, and maintenance status.
- **FAIR** principles for scientific data management.
- **Provenance is traceable.** A dedicated `Metadata` table records source, terminology, NLP system, and quality metrics for any attribute — essential when part of the data is extracted by NLP.

---

## Repository contents

| Artifact | File | Purpose |
| --- | --- | --- |
| **Minimum Data Set (MDS)** | `OSIRIS_RWD_Minimum_DataSet__MDS__*.xlsx` | The authoritative variable list: tables, variables, definitions (FR/EN), mandatory status, expected types/formats, value sets and terminologies. |
| **Data population guide** | `OSIRIS_RWD_Guide_d_alimentation_*.docx` | Per-variable population rules, edge cases, examples, generic rules, and annexes. (Currently in French.) |
| **Physical model** | `OSIRIS_RWD_Physical_model_*.png` | Entity-relationship diagram (keys, cardinalities, foreign keys). |
| **Changelog** | `OSIRIS_RWD_Changelog_*.xlsx` | Version-to-version changes, each tagged with a GitHub issue number and a **retro-compatibility impact** flag. |
| **REDCap data dictionary** *(forthcoming)* | `redcap/OSIRIS_RWD_REDCap_DataDictionary_*.csv` | REDCap-importable definition of the model as an eCRF — instruments, fields, types, value sets, and branching logic. **Planned, not yet released.** |

> The REDCap data dictionary and its companion code lists are **in preparation**; see [Data capture with REDCap](#data-capture-with-redcap-forthcoming) below.

---

## Model at a glance

OSIRIS RWD is a **patient-centric, event-based star schema**.

- **`Patient` is the only mandatory link.** In real-world data, the patient is the sole relationship known with certainty; every other table attaches to `Patient` (directly or through a parent entity).
- **27 tables** in the physical model; **114 documented variables**, of which **73 are technically mandatory** (per the MDS).
- A `Patient` row is unique on `PatientID` and belongs to exactly one `Dataset`.

### Table catalogue (by domain)

**Metadata & provenance**
- `Dataset` — describes the data extract (origin center, name, derived update date). Unique on *(OriginCenterID, DatasetName)*.
- `Metadata` — describes any attribute in the base: source, vocabulary, NLP system / lexical variant / quality measure / score, and the `Parent{Concept,ID,Variable}` it attaches to.

**Patient & follow-up**
- `Patient` — birth date, biological sex, update date.
- `Opposition` — patient information & objection to secondary data reuse (one row per patient).
- `LastNews` — vital status, death/last-visit/last-contact dates (one row per patient).

**Risk & history**
- `RiskFactor` / `Infection` — IARC Group 1 carcinogens and oncogenic infectious agents.
- `FamilyCancerHistory` — 1st/2nd-degree family cancer history.
- `RelatedPathology` — medical history, comorbidities, and adverse events (ICD-10), merged into one item.

**Cancer**
- `PrimaryCancer` — topography/morphology (ICD-O-3.2), groups, laterality, diagnosis date/method. Tumor uniqueness follows IACR rules on *(TopographyGroup, MorphologyGroup, Laterality)*.
- `TumorSize`, `TNMEvent`, `PrimaryCancerStage`, `PrimaryCancerGrade`.
- `TumorPathoEvent` (tumor events: relapse/progression) and `Metastasis`.
- `Progression`.

**Treatments**
- `Surgery` — surgery & interventional radiology procedures (CCAM).
- `Medication` — systemic therapies (ATC), clinical-trial / early-access flags.
- `Radiotherapy` / `RadiatedOrgan`.

**Specimens, biomarkers & exams**
- `BiologicalSpecimen`, `Biomarker` (incl. HGNC/UniProt gene & protein references).
- `Analysis`, `ImagingSeries` (DICOM-aligned imaging/nuclear-medicine).

**Generic extension tables**
- `Measure` and `Observation` — generic numeric/qualitative facts attached to any table row via `Parent{Concept,ID}`, terminology-flexible (SNOMED CT / NCIt / LOINC depending on the concept).

The clinical core can be extended with additional modules (e.g. genomic, radiomic) where needed.

---

## Generic rules (must-read before implementing)

- **Dates are split into three attributes** (day / month / year). When a collected date is imprecise, leave the missing parts empty — **except the year, which is always required**. (The eCRF imputation convention of `15`/`06` applies *only inside the eCRF* and must remain distinguishable from real data.)
- **Missing values are left empty.** Only explicit data is collected; there is no imputation or computation of missing information. Non-applicable vs unknown is distinguished only in the eCRF (NA / UK).
- **Update semantics.** All information for a patient is updated together; `PatientUpdateDate` carries the timestamp. `DatasetUpdateDate` is derived as the max over the dataset's patients.
- **Pseudonymisation is the implementer's responsibility.** The local IPP is directly identifying; dates of birth/care/death are quasi-identifiers. Each project using the model is responsible for regulatory compliance of secondary use (pseudonymisation, patient information, objection handling). See the EDPB pseudonymisation guidance.
- **Terminologies.** When a terminology is specified for an item, the expected values are codes from that terminology, formatted as `[terminology]_[version]` in `Metadata.Vocabulary`.

---

## Data capture with REDCap

A **REDCap data dictionary (CSV) and project template (XML)** are available in this repository under `redcap/`.
They provide the REDCap-native, importable definition of OSIRIS RWD as an electronic case report form (eCRF), intended primarily to **assess the quality of automatically structured data** and to support manual capture where needed.

It provides:

- **One REDCap instrument per OSIRIS RWD table**, with fields mapping 1:1 to MDS variables (matching variable names, types, and value sets).
- **Field-level validation** (date parts as day/month/year, numeric ranges, formats such as 9-digit FINESS) and **terminology-backed choice lists** (e.g. ICD-O-3.2, ICD-10, ATC, CCAM) exported as REDCap code lists.
- **Branching logic and repeating instruments** to reflect model cardinalities (e.g. 0..n primary cancers, treatments, biomarkers per patient).
- **eCRF-specific conventions** that differ from the database rules — notably the imprecise-date imputation (`15` for day, `06` for month) used *only inside the eCRF*, and the explicit **NA / UK** distinction for non-applicable vs unknown values, both of which must remain identifiable as eCRF artifacts rather than source data.

### Deployment & Installation

1. **Project Creation**: Create a new REDCap project using the provided XML file.
2. **Data Dictionary Import**: Log in with administrator privileges and import the Data Dictionary (`.csv` format) into the newly created project.
3. **Center Configuration**:
   - Update the `origincenterid` item value in the **Patient** form to match the participating center's FINESS code (9 digits).
   - *Single-center setup*: You can set the default FINESS code using the REDCap Action Tag on the item:
     ```text
     @DEFAULT='finess_code'
     ```
     *(replace `finess_code` with the center's 9-digit FINESS code).*


The data dictionary will be versioned and changelogged alongside the rest of the model; breaking changes affecting it are flagged in the changelog like any other artifact.

---

## Terminologies & referentials

The MDS value sets reference, among others:

`ICD-O-3.2` (topography & morphology) · `ICD-10` · `ATC` · `CCAM` · `LOINC` · `SNOMED CT` · `NCIt` · `HGNC` · `UniProt` · `UCUM` · `DICOM` · `FINESS` (center identification) · `ENCR`.

Codes intended for the French context default to HIS-aligned referentials (FINESS, CCAM, ICD-10/PMSI) with ANS recommendations and crosswalk availability as selection criteria.

---

## Versioning

OSIRIS RWD follows semantic versioning (`MAJOR.MINOR.PATCH`). The current release is **2.0.0**.

Every documented change is tracked against a **GitHub issue** and flagged for **retro-compatibility impact** in the changelog. Changes touching the physical model, MDS, data population guide, or REDCap material are listed there. When implementing, always read the changelog for the version you are migrating from — several `v2.0.0` changes (e.g. composite `Dataset` key, mandatory `Opposition`, removal of "mandatory status" items, `Imaging` → `ImagingSeries` rename) are **breaking**.

---

## Repository structure (TO BE VALIDATED)

```
.
├── README.md
├── CHANGELOG.md                # or the .xlsx changelog, summarized
├── mds/                        # Minimum Data Set (xlsx + exported CSV/value sets)
├── data-population-guide/      # data population guide (docx / PDF / md)
├── physical-model/             # ER diagram (png/svg) + DDL if available
├── redcap/                     # REDCap data dictionary (CSV) + code lists — forthcoming
└── modules/
    ├── genomic/
    └── radiomic/
```

---

## Contributing

Contributions follow an **open, issue-based process**: propose new concepts or updates to existing ones via [GitHub issues](https://github.com/InstitutNationalduCancer/OSIRIS_WG_OSIRIS-RWD/issues). Each accepted change is recorded in the changelog with its issue number and retro-compatibility impact. (Document your local contribution/RFC and conformance policy in `CONTRIBUTING.md` / `GOVERNANCE.md`.)

---

## Citation (TO BE VALIDATED)

A reference publication for OSIRIS RWD is **forthcoming**. Until it is available, please cite the model directly, e.g.:

> OSIRIS RWD — Common Data Model for Real-World Oncology Data, version 2.0.0 (2026). INCa project, coordinated with the support of Unicancer. Available at: <https://github.com/InstitutNationalduCancer/OSIRIS_WG_OSIRIS-RWD>.

This section will be updated with the full citation once the publication is released.

---

## License  (TO BE VALIDATED)

OSIRIS RWD is released under the **Creative Commons Attribution-ShareAlike 4.0 International License (CC BY-SA 4.0)**.

You are free to share and adapt the material, including for commercial purposes, provided that you (1) give appropriate **attribution** and (2) distribute any adapted material under the **same license** (ShareAlike). Note that CC BY-SA also covers *Sui Generis* Database Rights, which is appropriate for a data model and its value sets.

See the [`LICENSE`](LICENSE) file for the full text, or <https://creativecommons.org/licenses/by-sa/4.0/>.

> **Attribution example:** "OSIRIS RWD", an INCa project coordinated with the support of Unicancer, licensed under CC BY-SA 4.0.

---

## Acknowledgements

OSIRIS RWD is a project of the **Institut National du Cancer (INCa)**, coordinated with the support of **Unicancer**. The model was defined by the OSIRIS RWD Working Group, gathering members from comprehensive cancer centres, university hospitals, and partner organizations (full list in the data population guide annex).
