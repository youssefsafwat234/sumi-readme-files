# Feature 17 — Exam import & sharing · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-17-exam-import-sharing.md](../features/feature-17-exam-import-sharing.md) · mobile guide: [feature-17-exam-import-sharing-mobile.md](feature-17-exam-import-sharing-mobile.md)

---

## 1. Scope

* Excel template, upload, parsing, preview, commit (bank and exam), expiry and pruning.
* Google Form quiz import (behind a server flag).
* Bank questions into an exam; sharing exams; using a bank exam.
* Admin panel: *Question imports* (read-only).

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
php artisan queue:work          # Excel parsing (or QUEUE_CONNECTION=sync)
```

* «امتحان الوحدة الأولى» is **shared** in the bank.
* A committed history import «unit2-questions.xlsx»: 24 rows, 22 valid (row 7 «مفيش إجابة صحيحة متعلّمة», row 15 «MCQ2»).
* Fixtures `postman/fixtures/`: `valid-questions.xlsx`, `unit2-questions.xlsx`, `formulas.xlsx`, `wrong-headers.xlsx`, `too-many-rows.xlsx`.
* Google Forms needs `DARSLY_GOOGLE_FORMS_ENABLED=true` and a real access token; otherwise 503.

Postman: `postman/Darsly-Feature-17-Exam-Import-Sharing.postman_collection.json`.

---

## 3. Test cases: Excel

| ID | Case | Expected |
|---|---|---|
| IX-01 | Download the template | RTL sheet, Arabic headers, type dropdown, one example per type |
| IX-02 | Re-upload the untouched template | its examples are valid |
| IX-03 | `valid-questions.xlsx` | 202 `parsing` → `ready`; all valid |
| IX-04 | `unit2-questions.xlsx` | 24 / 22 valid; rows 7 and 15 with their codes and Arabic messages |
| IX-05 | `?valid=0` / `?valid=1` | only invalid / only valid rows |
| IX-06 | Commit | `{created: 22}`; questions in the bank; invalid rows ignored |
| IX-07 | Commit again | 409 `IMPORT_ALREADY_COMMITTED` |
| IX-08 | Upload with `exam_id`, commit | questions also in the exam (`added_to_exam`) |
| IX-09 | Commit into an exam with attempts | 409 `EXAM_HAS_ATTEMPTS`; nothing created |
| IX-10 | `wrong-headers.xlsx` | 422 `IMPORT_FILE_INVALID`, `reason: wrong_headers`, `missing_headers` |
| IX-11 | A `.csv` renamed `.xlsx` / a pdf | 422 `IMPORT_FILE_INVALID` (`not_xlsx`) |
| IX-12 | `too-many-rows.xlsx` | 422 `IMPORT_TOO_MANY_ROWS` |
| IX-13 | `formulas.xlsx` | formulas not evaluated (cached value or empty) |
| IX-14 | Unknown category name / with `create_missing_categories=1` | row error `UNKNOWN_CATEGORY` / category created on commit |
| IX-15 | The same question twice | warning `DUPLICATE_QUESTION`, still valid |
| IX-16 | All rows invalid → commit | 422 `IMPORT_NOTHING_VALID` |
| IX-17 | Preview while parsing / failed | 409 `IMPORT_NOT_READY` |
| IX-18 | Import older than 24 h | 410 `IMPORT_EXPIRED`; pruned by `darsly:imports:prune` |
| IX-19 | Import of another course | 404 `IMPORT_NOT_FOUND` |

## 4. Test cases: Google Form

| ID | Case | Expected |
|---|---|---|
| IG-01 | Flag off | 503 `GOOGLE_FORM_UNAVAILABLE` |
| IG-02 | A quiz form | 201 `ready`; radio → single, checkboxes → multiple, صح/خطأ → true_false; points and feedback mapped |
| IG-03 | Short answer / grid items | invalid rows «نوع السؤال ده مش مدعوم» |
| IG-04 | Not a quiz | 422 `GOOGLE_FORM_NOT_QUIZ` |
| IG-05 | Another person's form / no scope | 403 `GOOGLE_FORM_NOT_ACCESSIBLE` |
| IG-06 | A non-forms link | 422 `GOOGLE_FORM_INVALID_URL` |

## 5. Test cases: bank & sharing

| ID | Case | Expected |
|---|---|---|
| IB-01 | `import-from-bank` 3 questions | added to the exam (reused, no copies) |
| IB-02 | Bank exams list | «امتحان الوحدة الأولى» card |
| IB-03 | Use it | 201 new draft «… · نسخة», same settings and questions, no groups / dates; checklist missing groups |
| IB-04 | Use an exam that isn't shared | 404 `EXAM_NOT_FOUND` |
| IB-05 | Share / unshare via `PATCH {shared_in_bank}` | appears / disappears in the list |

## 6. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Course → Question imports | source, status, rows / valid / invalid, failure reason; read-only |

## 7. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Polling stops at `ready` / `failed`. |
| MB-02 | Invalid rows readable in Arabic and English. |
| MB-03 | Google Form option hidden on 503. |

## 8. Bug report

Course / exam / import ids, the uploaded file, request and full response (`error_code`, `details`, row errors), the test ID and the time.
