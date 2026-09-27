# Agent Instructions: Workspace Setup, Data Preparation & Submission Compliance

**Project:** Business Entity Resolution Challenge
**Audience:** An execution agent working in this repository. Follow every instruction literally and in order. Do not skip steps. Do not use any external data source, API, or lookup service at any point — see the Fair Play section, violation causes disqualification.

---

## 0. Current State

The workspace root currently contains four folders:

```
workspace/
├── code/              (empty — team's pipeline code goes here)
├── output/            (empty — final result files go here)
├── dataset/           (empty or partially filled — final clean dataset files go here)
└── student_resource/  (the organizer-provided package, freshly extracted — contains the real files)
```

`student_resource/` is the source of truth for raw data and the organizer's tooling. Nothing inside it should be edited in place — only copied/moved out to the correct destination, per Section 1.

---

## 1. Target Folder Structure

Reorganize the workspace until it matches this exactly. This structure must be preserved through to final submission packaging (Section 6).

```
workspace/
├── dataset/
│   ├── raw/
│   │   ├── train/
│   │   │   ├── train_source1.tsv
│   │   │   ├── train_source2.tsv
│   │   │   ├── train_source3.tsv
│   │   │   └── train_ground_truth.tsv
│   │   └── test/
│   │       ├── test_source1.tsv
│   │       ├── test_source2.tsv
│   │       └── test_source3.tsv
│   └── clean/
│       ├── train/          (cleaned versions of the 4 train files, same filenames)
│       ├── test/            (cleaned versions of the 3 test files, same filenames)
│       ├── val/              (entity-level validation split, built from train)
│       └── cleaning_log.md   (audit record — see Section 3, Step 8)
├── utils/
│   └── validate_submission.py     (copied unmodified from student_resource/utils)
├── code/
│   └── business_entity_resolution/
│       ├── src/                        (all pipeline source code)
│       ├── README.md                   (how to reproduce end-to-end)
│       └── requirements.txt            (pinned dependency versions)
├── output/
│   ├── matching_results.tsv       (final leaderboard submission file)
│   └── candidate_pairs.tsv        (blocking stage output, pre-model)
└── Documentation_template.md      (copied from student_resource, then filled in)
```

Note the split of `dataset/` into `raw/` (untouched, exactly as received) and `clean/` (output of the cleaning process). Never overwrite a raw file in place — always write cleaned output to a new file under `clean/`. This preserves an audit trail and lets you re-run cleaning without re-extracting the zip.

---

## 2. Reorganization Steps

1. Delete any `__MACOSX` folder and every `.DS_Store` file anywhere in the workspace — these are macOS zip artifacts, not real data. Search recursively and remove all of them.
2. Inside `student_resource/`, locate the `dataset/train/` and `dataset/test/` folders. Copy their contents into `dataset/raw/train/` and `dataset/raw/test/` respectively (create these folders if they don't exist). Do not rename any of the `.tsv` files.
3. Locate `student_resource/utils/` (contains `validate_submission.py`). Copy it, unmodified, to a new `utils/` folder at the workspace root.
4. Locate `student_resource/Documentation_template.md`. Copy it, unmodified, to the workspace root.
5. Locate `student_resource/README.md`. This is the organizer's own instructions file, not your team's. Copy it to the workspace root under a distinct name, e.g. `ORGANIZER_README.md`, so it is never confused with or overwritten by the team's own `code/business_entity_resolution/README.md` (which documents your own pipeline, written later).
6. Verify every file arrived intact: confirm `dataset/raw/train/` contains exactly 4 files, `dataset/raw/test/` contains exactly 3 files, and `utils/validate_submission.py` exists and is non-empty.
7. Once verified, delete the now-redundant `student_resource/` folder entirely. Do not delete it before verification.
8. Create the remaining empty target folders that don't exist yet: `dataset/clean/train/`, `dataset/clean/test/`, `dataset/clean/val/`, `code/business_entity_resolution/src/`.
9. Confirm the workspace now matches Section 1 exactly, with no leftover folders (`student_resource`, `__MACOSX`, extraction artifacts) anywhere.

---

## 3. Data Cleaning Requirements (Process Only — Implement in Code Separately)

The evaluation metric (F_0.5, defined in Section 5) penalizes false merges twice as heavily as missed matches. This means cleaning must **standardize noisy formatting without ever erasing information that distinguishes two genuinely different businesses.** Over-aggressive cleaning that makes distinct entities look identical will directly hurt the score. Every step below must be applied identically and deterministically to all six source files (`train_source1/2/3`, `test_source1/2/3`) so that train and test data are processed through the exact same logic.

1. **Never touch `entity_id`.** This column must pass through cleaning completely unmodified — no casing changes, no whitespace trimming beyond what's already correct. Altering an ID breaks every downstream join and can cause submission rejection.
2. **Whitespace normalization.** Strip leading/trailing whitespace from `business_name`, `business_address`, and `country`. Collapse any run of multiple spaces (including non-standard Unicode spaces, tabs accidentally embedded in a field, etc.) into a single regular space.
3. **Unicode normalization.** Normalize all text fields to a consistent Unicode form so that visually identical characters (e.g. different dash characters, curly vs straight quotes, combined vs decomposed accented characters) compare equal. Apply this cautiously to transliterated names — do not strip diacritics or non-Latin characters that carry real distinguishing signal; only normalize equivalent representations of the same character.
4. **Casing.** Lowercase `business_name` and `business_address` for matching purposes. Keep the original-case value alongside the cleaned value (a separate column, not an overwrite) so a human reviewer or the documentation can reference the original text later.
5. **Controlled abbreviation expansion — word-boundary matching only.** Expand well-known abbreviations so that `"Pvt Ltd"` and `"Private Limited"` are treated as equivalent, and similarly for `"St"` → `"Street"`, `"Rd"` → `"Road"`, `"Ave"` → `"Avenue"`, `"&"` → `"and"`, `"Corp"` → `"Corporation"`, `"Inc"` → `"Incorporated"`. Every expansion must use strict word-boundary logic so that a substring inside a longer word (e.g. the "st" inside "street" or "fastest") is never accidentally rewritten. Do not invent additional abbreviation rules beyond well-established, unambiguous ones — an incorrect expansion is itself a source of false merges.
6. **Preserve address structure.** Do not strip commas or reorder components inside `business_address` during cleaning — component order and separators are themselves useful signal for matching and must be left intact for the matching stage to use. Cleaning here means fixing formatting noise (case, whitespace, abbreviations), not restructuring the address.
7. **Do not drop or impute missing/blank records.** Every Source 1 entity — including ones with a blank or missing `business_name`/`business_address` — must still appear in the final cleaned file and ultimately in the submission (Constraint 3, Section 5). If a field is blank, leave it blank rather than guessing a value or removing the row.
8. **Keep a cleaning audit log.** For every file processed, record: total rows in, total rows out (should always match), how many `business_name`/`business_address` values were altered by abbreviation expansion, and how many were altered by Unicode/whitespace normalization only. Save this as `dataset/clean/cleaning_log.md`. This feeds directly into the required methodology write-up.
9. **No external augmentation of any kind.** Cleaning must be self-contained — no dictionary, gazetteer, geocoding call, or business registry lookup from outside the provided files. See Section 7; this is a disqualification-level violation, not a style preference.

---

## 4. Country Field — Mandatory Handling Rule

The training data contains only `US` and `India`. The test data additionally contains `France`, which never appears during training.

- Treat `country` as an **opaque string** used only to partition candidate matching (a Source 1 record can only be compared against Source 2/3 records sharing the identical `country` string). Do not filter, one-hot encode, hard-code, or branch logic on specific country values anywhere in the pipeline (cleaning, blocking, or modeling).
- Do not write conditional logic of the form "if country equals a specific value, do X" anywhere in the codebase. If such logic exists, it is a bug and must be removed — it will fail silently or crash the moment France is encountered.
- After cleaning, verify: every distinct value in the `country` column across all six files is handled correctly, `France` appears only in the test files, and no record has been dropped or misassigned during country-based partitioning.

---

## 5. Submission Format Constraints (from the Problem Statement — Mandatory)

These are hard requirements. Any violation causes the submission to fail validation and go unscored — there is no partial credit for a malformed file.

- **File format:** Both `matching_results.tsv` and `candidate_pairs.tsv` must be tab-separated, matching the exact column names specified by the organizers (`source1_entity_id`, `matched_entity_ids` / `candidate_entity_ids`).
- **Complete coverage:** Every Source 1 entity in the test set must appear as exactly one row in `matching_results.tsv`. A missing entity causes rejection.
- **No self-matches / no foreign IDs:** `matched_entity_ids` may only contain Source 2 or Source 3 IDs that exist in the test set. Any Source 1 ID, or any ID not present in the test set, causes rejection.
- **No duplicates:** No entity ID may appear twice within a single row's ID list. No `source1_entity_id` may appear as a duplicate row.
- **Singletons:** Leave `matched_entity_ids` empty (not `"None"`, not `"NaN"`, not a blank space) for any Source 1 entity with no matches.
- **`candidate_pairs.tsv` consistency:** Every ID that appears in `matching_results.tsv` must also appear in `candidate_pairs.tsv` for that same Source 1 entity — a match that was never a candidate signals a pipeline bug.
- **Metric:** Scoring is macro-averaged F_0.5 per Source 1 entity: `F_0.5 = (1.25 × Precision × Recall) / (0.25 × Precision + Recall)`. A correctly predicted empty list for a true singleton scores 1.0; a false match on a true singleton scores 0.0. Precision is weighted twice as heavily as recall — when in doubt during threshold tuning, favor rejecting a marginal match over accepting one.
- **Always validate before any leaderboard upload:** Run `utils/validate_submission.py` against `output/matching_results.tsv` and `output/candidate_pairs.tsv` and confirm it prints `PASS` (exit code 0) before submitting. Treat a non-zero exit code as a hard blocker.

---

## 6. Final Submission Package Constraints

The final zip archive (separate from the live leaderboard upload) must match this structure exactly:

```
<team_name>_submission.zip
├── output/
│   ├── matching_results.tsv
│   └── candidate_pairs.tsv
├── code/
│   └── business_entity_resolution/
│       ├── src/
│       ├── README.md
│       └── requirements.txt
└── Documentation_template.md
```

- `code/business_entity_resolution/` must be self-contained and runnable: anyone should be able to regenerate both output files from the raw train/test data using only what's inside this folder.
- `requirements.txt` must pin exact dependency versions (not loose ranges).
- `README.md` must give exact, literal run instructions: data → cleaning → blocking → matching → output.
- `Documentation_template.md` must be filled in (not left as a blank template) and must describe: methodology, candidate generation/blocking strategy, model architecture and feature engineering, and any other relevant details. There is no page limit; prioritize clarity and depth over brevity.

---

## 7. Fair Play & Model Constraints (Zero Tolerance)

- **No external data lookup of any kind, at any pipeline stage.** This explicitly includes: commercial entity resolution APIs/services, government business registry lookups, geocoding APIs for address normalization, and any data augmentation pulled from the internet. This applies to cleaning, blocking, and modeling alike. Any evidence of this results in immediate disqualification, not a score penalty.
- **Model constraint:** The final matching model must be licensed under MIT or Apache 2.0, and must not exceed 8 billion parameters. Verify the license and parameter count of any pretrained model before adopting it into the pipeline.
- All code, methodology, and pipelines are subject to review — the top-ranked teams' submission packages are audited in detail before final rankings are confirmed. Every design decision should be one the team can explain and defend if asked.

---

## 8. Completion Checklist

Before considering setup/cleaning "done," confirm all of the following:

- [ ] `student_resource/`, `__MACOSX`, and all `.DS_Store` files are gone
- [ ] `dataset/raw/train/` has all 4 original files, `dataset/raw/test/` has all 3, untouched
- [ ] `dataset/clean/train/`, `dataset/clean/test/` contain cleaned versions with identical filenames and row counts to the raw files
- [ ] `dataset/clean/val/` contains an entity-level split (built from `train`, not `test`) with zero entity-ID overlap between train and val
- [ ] `dataset/clean/cleaning_log.md` exists and documents what was changed and how much
- [ ] `utils/validate_submission.py` is present at the workspace root, unmodified
- [ ] `country` is handled as a plain string everywhere, with no hard-coded value comparisons anywhere in the codebase
- [ ] No step in the pipeline calls an external API, database, or internet data source
- [ ] `output/matching_results.tsv` and `output/candidate_pairs.tsv` exist and pass `validate_submission.py` with exit code 0
- [ ] `Documentation_template.md` is fully filled in, not left blank
- [ ] Final zip matches the exact structure in Section 6
