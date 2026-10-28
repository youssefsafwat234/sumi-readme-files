# Feature 09 — Course categories · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-09-categories.md](../features/feature-09-categories.md) · mobile guide: [feature-09-categories-mobile.md](feature-09-categories-mobile.md)

---

## 1. Scope

* Categories of a course: create, create several, list with counts, details, items per type, edit, reorder, merge, delete.
* Content linking (lectures, questions, exams, files): a category of the same course only.
* Student chips.
* Access: the course's teacher or a linked secretary; nobody else.
* Admin panel: Courses → a course → *Categories* tab.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

Password `Darsly#User2026`. On Mona's «مراجعة الرياضيات - الصف الثالث الثانوي»: **Supply**, **Production**, **أخرى**, **مراجعة نهائية** (empty), **الوحدة الأولى / الثانية / التالتة** (the question bank units, Feature 15) — 7 in total.

Postman: `postman/Darsly-Feature-09-Categories.postman_collection.json` (re-runnable; folder *Cleanup* last).

---

## 3. Test cases: create & edit

| ID | Case | Body | Expected |
|---|---|---|---|
| CC-01 | Create | `{name: "الوحدة الرابعة", color: "#1F6FEB"}` | **201**, last in the order, counts 0 |
| CC-02 | Same name, other case / extra spaces | `"supply"`, `"  Supply "` | 422 `CATEGORY_NAME_TAKEN` (`name`) |
| CC-03 | Name of a **deleted** category | delete, then create the same name | 201 |
| CC-04 | Same name in **another** course | | 201 |
| CC-05 | Name rules | empty, 61 characters | 422 (`name`) |
| CC-06 | Missing colour | `{name}` | 422 (`color`) |
| CC-07 | 51st category | | 422 `CATEGORY_LIMIT_REACHED` |
| CC-08 | Suggestions | 3 new names | 201, 3 created in order |
| CC-09 | Suggestions with one taken name / 11 names | | 422 (`names.N` / `names`); **nothing** created |
| CC-10 | Rename to a taken name | | 422 `CATEGORY_NAME_TAKEN` |
| CC-11 | Recolour only | `{color}` | name unchanged |

## 4. Test cases: list, counts, items

| ID | Case | Expected |
|---|---|---|
| CL-01 | List the math course | 7 categories, `counts {lecture, question, exam, file, total}` |
| CL-02 | Counts after linking content | link a lecture → `counts.lecture` +1 |
| CL-03 | `GET /teacher/categories` (no course) | every course of Mona, grouped, each with `course_id` |
| CL-04 | `GET /teacher/categories?course_id=<Youssef's course>` | 403 `COURSE_NOT_ACCESSIBLE` |
| CL-05 | Items `?type=question` on «الوحدة الأولى» | a page of its questions |
| CL-06 | Items `?type=video` | 422 (`type`) |
| CL-07 | Category of another course in this course's URL | 404 `CATEGORY_NOT_FOUND` |

## 5. Test cases: order, merge, delete

| ID | Case | Expected |
|---|---|---|
| CO-01 | Reorder with the full list | 200; the list follows the new order |
| CO-02 | Missing / extra / duplicate / foreign id | 422 (`category_ids`); order unchanged |
| CM-01 | Merge Production into Supply | `moved` counts; Production gone; its content now in Supply |
| CM-02 | Merge into itself | 422 `CATEGORY_MERGE_SAME` |
| CM-03 | Merge into a category of another course | 404 / 422; nothing moved |
| CD-01 | Delete «أخرى» | `released` counts; content kept with `category: null` |
| CD-02 | Delete twice | 404 `CATEGORY_NOT_FOUND` |

## 6. Test cases: content linking

| ID | Case | Expected |
|---|---|---|
| CI-01 | Create a lecture / question / exam / file with a category of this course | saved with `category {id, name, color}` |
| CI-02 | …with a category of another course or a deleted one | 422 `INVALID_CATEGORY` (`category_id`) |
| CI-03 | `category_id: null` | uncategorized |

## 7. Test cases: student & access

| ID | Case | Expected |
|---|---|---|
| CS-01 | Nour (enrolled in math) | `GET /student/courses/{math}/categories` → `[{id, name, color}]` in order |
| CS-02 | A student not enrolled / without access | 404 `COURSE_NOT_FOUND` |
| CA-01 | Ahmed (secretary) creates | 201, belongs to Mona's course |
| CA-02 | Youssef on Mona's course | 403 `COURSE_NOT_ACCESSIBLE` |
| CA-03 | Mona with an expired platform subscription (Feature 23) | writes 402 `PLATFORM_SUBSCRIPTION_EXPIRED`; reads OK |

## 8. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Course → Categories tab | list with content counts |
| AD-02 | Create / rename / merge / delete | same rules and errors as the API (red notification on failure) |
| AD-03 | Support admin | read-only |

## 9. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Chips use the returned colour. |
| MB-02 | Drag-to-reorder sends the full list. |
| MB-03 | Delete confirmation shows the counts and says content is kept. |

## 10. Bug report

Account, `X-Teacher-Id`, course and category ids, request and full response (`error_code`, `errors`), the test ID and the time.


### Move one category

Send `category_id` and 1-based `position`. Verify moving last to first and first to last shifts others and saves the returned order. Reject missing/malformed IDs, foreign/deleted categories, missing/non-integer/zero/above-count positions, and combined legacy/new formats with 422 and no changes. Legacy full-list requests remain supported.
