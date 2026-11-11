# Ranker

A JSON-only content repository that powers an exam-preparation app. The app reads everything it shows (syllabus, books, videos, past papers, mock tests) directly from this repo over raw GitHub URLs, so there is no backend to run: **a `git push` is a content release.**

Currently one exam is listed: **RRB Station Master (Graduate Level) Single Stage CBT**. The layout is built so more exams can be added the same way.

---

## Contents

- [How the app uses this repo](#how-the-app-uses-this-repo)
- [Repository layout](#repository-layout)
- [Raw URLs](#raw-urls)
- [Data formats](#data-formats)
- [Station Master at a glance](#station-master-at-a-glance)
- [Mock tests](#mock-tests)
- [Known data issues](#known-data-issues)
- [Adding content](#adding-content)
- [Validating changes](#validating-changes)
- [Conventions](#conventions)

---

## How the app uses this repo

The app only ever needs raw file URLs. The fetch order is:

```
1. index.json                                  -> list of exams + where mock tests live
2. station_master/station_master_exam_resources.json   -> syllabus, books, videos, links, tips
3. station_master/mock_tests/index.json        -> list of every mock test file
4. station_master/mock_tests/mock_test_NNN.json -> one test, fetched when the user opens it
```

Steps 1 to 3 are small and can be cached. Step 4 is fetched on demand, so the app never downloads all 62 tests at once.

Because the app depends on these paths, **moving or renaming a file is a breaking change** for any installed app version that still uses the old URL. See [Conventions](#conventions).

---

## Repository layout

```
ranker/
├── index.json                                   Root index: all exams + mock test pointer
└── station_master/
    ├── station_master_exam_resources.json       Exam info, syllabus, books, videos, links
    └── mock_tests/
        ├── index.json                           List of every mock test (id, file, raw_url)
        ├── mock_test_001.json
        ├── mock_test_002.json
        ├── ...
        └── mock_test_062.json
```

| Path | Purpose |
|------|---------|
| `index.json` | Entry point. Lists exams under `resources` and points to the mock test index under `mock_tests`. |
| `station_master/station_master_exam_resources.json` | Everything that is not a test: notification facts, eligibility, pattern, syllabus topics, books, YouTube links, past papers, study notes, courses, tips. |
| `station_master/mock_tests/index.json` | The list of all mock test files with their ids and raw URLs. |
| `station_master/mock_tests/mock_test_NNN.json` | One full test: metadata, sections, questions, options, answer keys. |

---

## Raw URLs

Base: `https://raw.githubusercontent.com/VikasNadcab/ranker/main/`

| What | URL |
|------|-----|
| Root index | `.../main/index.json` |
| Station Master resources | `.../main/station_master/station_master_exam_resources.json` |
| Mock test index | `.../main/station_master/mock_tests/index.json` |
| A mock test | `.../main/station_master/mock_tests/mock_test_001.json` |

In `station_master/mock_tests/index.json`, `raw_url_base` already ends with `mock_tests/` and each `file` is a bare file name. Build a URL as `raw_url_base + file` (or just use the ready-made `raw_url`). Do not add a folder prefix on top.

---

## Data formats

### Root `index.json`

```jsonc
{
  "repo": "VikasNadcab/ranker",
  "repo_url": "https://github.com/VikasNadcab/ranker",
  "description": "...",
  "last_updated": "2026-11-11",
  "resources": [                       // one entry per exam
    {
      "id": "station_master",
      "name": "RRB Station Master (Graduate Level) Single Stage CBT",
      "category": "Railway Recruitment",
      "notification": "CEN 08/2026",
      "conducting_body": "Railway Recruitment Board (RRB)",
      "vacancies": 1265,
      "file": "station_master/station_master_exam_resources.json",
      "raw_url": "https://raw.githubusercontent.com/.../station_master_exam_resources.json",
      "description": "...",
      "contents": ["exam_pattern", "books", "..."],
      "last_updated": "2026-11-10"
    }
  ],
  "future_additions": "How to add another exam",
  "mock_tests": {
    "total_created": 62,
    "folder": "station_master/mock_tests/",
    "index_url": ".../station_master/mock_tests/index.json",
    "raw_url_base": ".../station_master/mock_tests/",
    "pattern": "Description of the standard test layout"
  }
}
```

### `station_master/mock_tests/index.json`

```jsonc
{
  "exam": "RRB Station Master (Graduate Level) Single Stage CBT",
  "notification": "CEN 08/2026",
  "total_tests": 62,
  "folder": "station_master/mock_tests/",
  "raw_url_base": "https://raw.githubusercontent.com/VikasNadcab/ranker/main/station_master/mock_tests/",
  "last_updated": "2026-11-11",
  "tests": [
    {
      "id": "SM_MOCK_001",
      "file": "mock_test_001.json",
      "raw_url": "https://raw.githubusercontent.com/VikasNadcab/ranker/main/station_master/mock_tests/mock_test_001.json"
    }
  ]
}
```

`total_tests` must equal the length of `tests`, and every `id` must be unique.

### `mock_test_NNN.json`

```jsonc
{
  "mock_test_id": "SM_MOCK_001",
  "exam": "RRB Station Master (Graduate Level) Single Stage CBT",
  "notification": "CEN 08/2026",
  "total_questions": 100,
  "total_marks": 100,
  "duration_minutes": 90,
  "marking_scheme": "+1 for correct, -1/3 for incorrect",
  "sections": [
    {
      "section": "Basic Science & Mathematics",
      "marks": 30,
      "questions": [
        {
          "q": 1,
          "question": "What is the speed of light in vacuum?",
          "options": ["3×10^8 m/s", "3×10^6 m/s", "3×10^7 m/s", "3×10^9 m/s"],
          "answer": "A",
          "topic": "Basic Science"
        }
      ]
    }
  ]
}
```

Field rules:

| Field | Rule |
|-------|------|
| `mock_test_id` | `SM_MOCK_` plus the three-digit number, matching the file name. |
| `q` | Question number, numbered 1..N across the whole test, not restarting per section. |
| `options` | Exactly 4 strings. |
| `answer` | One letter, `A` to `D`. `A` is `options[0]`, `B` is `options[1]`, and so on. |
| `topic` | Short topic label, useful for per-topic analysis. |
| `total_questions` / `total_marks` | Must equal the real number of questions and the sum of section marks. |

Scoring: +1 for a correct answer and -1/3 for a wrong one, as stated in `marking_scheme`. The files do not say how to treat unattempted questions; the app should score them 0.

### `station_master_exam_resources.json`

Top-level keys, in file order:

`exam_name`, `conducting_body`, `notification_number`, `vacancies`, `application_dates`, `eligibility`, `selection_process`, `pay_level`, `basic_pay`, `grade_pay`, `job_profile`, `exam_pattern`, `official_notification_pdf`, `official_syllabus_pdf`, `departmental_exam_syllabus`, `books`, `youtube_channels_and_videos`, `past_question_papers`, `mock_tests`, `study_notes`, `online_courses`, `important_links`, `preparation_tips`, `last_updated`, `data_sources`.

`exam_pattern` has `type`, `total_questions`, `total_marks`, `duration_minutes`, `marking_scheme`, `negative_marking`, `language`, and `sections`. Each entry in `sections` has a `section` name, its `marks`, and a list of syllabus `topics`.

Note that `mock_tests` inside this file lists **external** mock test providers (Testbook, Adda247) and is separate from the tests stored in this repo.

---

## Station Master at a glance

Taken from `station_master_exam_resources.json`. Always check the official notification before relying on dates.

| | |
|---|---|
| Exam | RRB Station Master (Graduate Level) Single Stage CBT |
| Notification | CEN 08/2026 |
| Conducting body | Railway Recruitment Board (RRB) |
| Vacancies | 1,265 |
| Applications | 27 October 2026 to 25 November 2026, at https://www.rrbapply.gov.in/ |
| Eligibility | Graduation in any discipline, age 20 to 33, medical standard A-2 |
| Selection process | CBT, then CBAT, then document verification, then medical examination |
| Pay | Level 6 (7th CPC), Rs. 35,400 to Rs. 1,12,400 |
| CBT | 100 questions, 100 marks, 90 minutes, +1 / -1/3 marking |

Sections in the official pattern, as recorded in the resources file: Basic Science & Mathematics (30), Data Analysis & Interpretation (30), Logical Capability (20), Mental Reasoning (20).

---

## Mock tests

62 tests, 5,920 questions in total. The tests do not all share one layout:

| Tests | Count | Sections (marks / questions) |
|-------|-------|------------------------------|
| 001 to 017, 022 to 038 | 34 | Basic Science & Mathematics 30/30, Data Analysis & Interpretation 30/30, Logical Capability 20/20, Mental Reasoning 20/20 |
| 018 to 021 | 4 | Basic Science & Mathematics 30/30 only (partial tests) |
| 039 to 062 | 24 | General Awareness 40/40, Mathematics 30/30, Logical Capability 20/20, Mental Reasoning 20/10 |

The first row is the standard layout and matches the official pattern above.

---

## Known data issues

These were found by validating the files and are **not fixed yet**. They affect what the app shows, so fix them before relying on the affected tests.

1. **Tests 018 to 021 are incomplete.** Each has 30 questions (one section) but declares `total_questions: 100`, `total_marks: 100` and `duration_minutes: 90`. The app would show a 100-question test that ends after 30.
2. **Tests 039 to 062 have inconsistent marks.** The `Mental Reasoning` section declares `marks: 20` but contains only 10 questions. Section marks therefore add up to 110 while `total_marks` is 100. Either the section needs 10 more questions or its `marks` should be 10.
3. **Tests 039 to 062 use a different section layout** (General Awareness and Mathematics instead of Basic Science & Mathematics and Data Analysis & Interpretation). Decide whether this is intended or whether they should follow the standard layout.
4. **Many questions repeat across tests.** There are 5,920 questions but only 1,917 distinct question texts, and 599 of those texts appear in more than one test. Tests overlap heavily, which weakens them as separate practice papers.

---

## Adding content

### Add a mock test to Station Master

1. Create `station_master/mock_tests/mock_test_063.json` following the [format](#mock_test_nnnjson), with `mock_test_id` set to `SM_MOCK_063`.
2. Append an entry to `tests` in `station_master/mock_tests/index.json`:
   ```json
   {
     "id": "SM_MOCK_063",
     "file": "mock_test_063.json",
     "raw_url": "https://raw.githubusercontent.com/VikasNadcab/ranker/main/station_master/mock_tests/mock_test_063.json"
   }
   ```
3. Update `total_tests` and `last_updated` in that index, and `mock_tests.total_created` and `last_updated` in the root `index.json`.
4. Run the [validation script](#validating-changes).

### Add a new exam

1. Create a folder named after the exam id, for example `ssc_cgl/`.
2. Add `ssc_cgl/ssc_cgl_exam_resources.json`, using the same top-level structure as the Station Master resources file where it applies.
3. Append an entry to `resources` in the root `index.json` with `id`, `name`, `category`, `notification`, `conducting_body`, `vacancies`, `file`, `raw_url`, `description`, `contents`, and `last_updated`.
4. If the exam has mock tests, follow the same `ssc_cgl/mock_tests/` plus `index.json` pattern, and extend the root `mock_tests` pointer (it currently describes only Station Master) so the app can find them.

---

## Validating changes

Run this from the repo root before every commit. It needs only Python 3.

```python
import json, glob, os

errors = []
base = "station_master/mock_tests/"
index = json.load(open(base + "index.json"))

listed = [t["file"] for t in index["tests"]]
actual = sorted(os.path.basename(p) for p in glob.glob(base + "mock_test_*.json"))
if sorted(listed) != actual:
    errors.append("index.json and the files on disk do not match")
if index["total_tests"] != len(listed):
    errors.append("total_tests does not equal the number of listed tests")
if len({t["id"] for t in index["tests"]}) != len(listed):
    errors.append("duplicate test ids in index.json")
for t in index["tests"]:
    if t["raw_url"] != index["raw_url_base"] + t["file"]:
        errors.append(f"{t['file']}: raw_url does not match raw_url_base + file")

for name in actual:
    d = json.load(open(base + name))
    qs = [q for s in d["sections"] for q in s["questions"]]
    if len(qs) != d["total_questions"]:
        errors.append(f"{name}: {len(qs)} questions, declares {d['total_questions']}")
    if sum(s["marks"] for s in d["sections"]) != d["total_marks"]:
        errors.append(f"{name}: section marks do not add up to total_marks")
    if [q["q"] for q in qs] != list(range(1, len(qs) + 1)):
        errors.append(f"{name}: question numbers are not 1..N")
    for q in qs:
        if len(q["options"]) != 4 or q["answer"] not in "ABCD":
            errors.append(f"{name} q{q['q']}: bad options or answer")

print("\n".join(errors) if errors else "all checks passed")
```

At the time of writing this reports the problems listed under [Known data issues](#known-data-issues).

---

## Conventions

- **Valid JSON only**, UTF-8, two-space indentation, one trailing newline.
- **Never rename or move published files** without a plan for older app versions. If a move is unavoidable, update both index files in the same commit and keep the old files in place until the app no longer requests them.
- **Keep indexes in sync** with the files in the same commit: file lists, counts, and `last_updated` dates.
- **Answers are letters**, not option text or numbers, and options are never reordered after a test is published.
- **Questions should be original or properly sourced.** Do not paste copyrighted question papers or books into the repo.
- **Dates** use `YYYY-MM-DD` in `last_updated` fields.
