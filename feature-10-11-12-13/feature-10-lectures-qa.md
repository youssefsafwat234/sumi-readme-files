# Feature 10 — Lectures · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-10-lectures.md](../features/feature-10-lectures.md) · mobile guide: [feature-10-lectures-mobile.md](feature-10-lectures-mobile.md)

---

## 1. Scope

* Lecture CRUD inside a course: number, title, cover, description, category, groups, publish window.
* Derived status (`draft`, `scheduled`, `published`, `ended`), lists and filters, group page, next number.
* Student visibility (published + student's group + access).
* Admin panel: Courses → a course → *Lectures* tab.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

On Mona's «مراجعة الرياضيات - الصف الثالث الثانوي»: Preface, Chapter 1 · Part 1, Chapter 1 · Part 2, Production practical, Chapter 2 · Part 1, Chapter 2 · Practical (**scheduled**), across Supply / Production and groups A, B, evening. Students: Omar, Nour (group A), Sara, Laila.

Postman: `postman/Darsly-Feature-10-Lectures.postman_collection.json` (folder *Create - FULL payload* sends every field).

---

## 3. Test cases: create

| ID | Case | Body | Expected |
|---|---|---|---|
| LC-01 | Minimal | `{title, group_ids:[A]}` | **201**; number = max + 1; `status: draft`; `next_lecture_number` = number + 1 |
| LC-02 | Full JSON | every field incl. `publish_now: true` | 201; `status: published` |
| LC-03 | Cover (multipart) | jpg 1 MB | `image_url` + `thumb_url`; 6 MB / pdf → 422 |
| LC-04 | Number taken | an existing number | 422 `LECTURE_NUMBER_TAKEN` |
| LC-05 | Number of a deleted lecture | | 201 (reusable) |
| LC-06 | `publish_now` + `published_at` | | 422 (`published_at`) |
| LC-07 | `available_until` before `published_at` | | 422 (`available_until`) |
| LC-08 | Future `published_at` | tomorrow | `status: scheduled` |
| LC-09 | No groups / group of another course | `[]` / foreign | 422 / 422 `INVALID_GROUP_ASSIGNMENT` |
| LC-10 | Inactive group | Physics summer group on its course | 201 (allowed; grants no access) |
| LC-11 | Category of another course | | 422 `INVALID_CATEGORY` |
| LC-12 | A failing nested block | a bad `conditions.required_lecture_ids` | 422; **no lecture created** |

## 4. Test cases: lists

| ID | Case | Expected |
|---|---|---|
| LL-01 | Course list | compact cards, newest number first, chips in `data.filters.categories` |
| LL-02 | `?status=scheduled` | Chapter 2 · Practical only |
| LL-03 | `?category_id=` / `?uncategorized=1` / both | filtered / filtered / 422 |
| LL-04 | `?search=Part 1` / `?search=2` | by title / by number |
| LL-05 | Group page A | only lectures assigned to A; `tabs {lectures, exams, files}` |
| LL-06 | `/teacher/lectures?group_id=A` | same cards; resolves the course from the group |
| LL-07 | `/teacher/lectures` (nothing) | every course of Mona, no chips |
| LL-08 | Next number | `{next_lecture_number}` = max + 1 |

## 5. Test cases: edit & delete

| ID | Case | Expected |
|---|---|---|
| LE-01 | Title only | other fields unchanged |
| LE-02 | `group_ids` [A, B] → [B] | A detached; students of A lose it |
| LE-03 | `published_at: null` | back to draft; hidden from students |
| LE-04 | Replace / remove the cover | new URL / `remove_image: 1` → `image_url: null` |
| LE-05 | Delete | 200; gone from lists; its number reusable |
| LE-06 | Another teacher's lecture | 403 / 404 |

## 6. Test cases: student visibility

| ID | Case | Expected |
|---|---|---|
| LS-01 | Nour (group A) lists math lectures | only published lectures of group A, in course order |
| LS-02 | Scheduled / ended / draft lecture | not listed; `GET /student/lectures/{id}` → 404 `LECTURE_NOT_FOUND` |
| LS-03 | Remove Nour from group A | its lectures disappear |
| LS-04 | Suspend Nour's enrollment / expired subscription | no lectures (`404 COURSE_NOT_FOUND` on the list when not enrolled) |
| LS-05 | Course access window closed (Feature 22) | 403 `COURSE_NOT_OPEN_YET` / `COURSE_ACCESS_CLOSED` |

## 7. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Course → Lectures tab | list with status and groups |
| AD-02 | Create / edit / delete | same rules; audit actor = admin |
| AD-03 | Support admin | read-only |

## 8. Mobile checks

| ID | Check |
|---|---|
| MB-01 | "Save & add another" keeps the form and pre-fills the next number. |
| MB-02 | Status badge matches `status`. |
| MB-03 | A failed save keeps every field (nothing was saved). |

## 9. Bug report

Account, `X-Teacher-Id`, course / lecture / group ids, request and full response, the test ID and the time (publish dates are time-sensitive).


### Exhausted view allowances

Teacher lecture details include `counts.exhausted_students`: number of lecture progress records with `views_used >= max_views + extra_views`. This counts exhausted allowances, not completed watching. Unlimited views (`max_views: null`) returns 0. Verify the exact limit, extra views, and unlimited access.
