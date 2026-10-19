# Feature 03 — Courses · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-03-courses.md](../features/feature-03-courses.md) · mobile guide: [feature-03-courses-mobile.md](feature-03-courses-mobile.md)

---

## 1. Scope

* Master-data pickers: study tracks, subjects of a track, academic years.
* Courses: create, list (filters, search, pagination), details, edit (fields and image), delete.
* Ownership: a course belongs to the teacher; a linked secretary works on it in that teacher's context.
* Admin panel:
  * *Academic data* (study tracks, subjects, academic years);
  * *Teaching → Courses* (every teacher's courses, managed on the teacher's behalf).

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

Password `Darsly#User2026` for every account; admin panel `/admin` with `admin@darsly.test`.

| Account | Role |
|---|---|
| `mona.mahmoud@darsly.test` | Teacher A |
| `youssef.ibrahim@darsly.test` | Teacher B |
| `ahmed.hassan@darsly.test` | Secretary of Mona (active link), suspended link with Youssef |
| `nour.salem@darsly.test` | Secretary of Mona (active), expired link with Youssef |

Seeded courses:

| Teacher | Course | Year / term |
|---|---|---|
| Mona | مراجعة الرياضيات - الصف الثالث الثانوي | 2026 / 2027 · first |
| Mona | شرح الفيزياء - الترم الأول | 2026 / 2027 · first |
| Mona | أساسيات الكيمياء | 2025 / 2026 · second |
| Youssef | اللغة الإنجليزية - مراجعة نهائية | 2026 / 2027 · second |
| Youssef | الأحياء - المنهج الكامل | 2026 / 2027 · first |
| Youssef | الرياضيات - الصف الثالث الإعدادي | 2025 / 2026 · first |

Seeded study tracks: *General Secondary* and *Preparatory Stage*, each with subjects. Academic years: *2025 / 2026* and *2026 / 2027*.

Headers for course calls: the token plus `X-Teacher-Id` = the **teacher role id** (from `data.user.roles` in the login response). Postman: `postman/Darsly-Feature-03-Courses.postman_collection.json`.

---

## 3. Test cases: master data (no `X-Teacher-Id`)

| ID | Case | Expected |
|---|---|---|
| MD-01 | `GET /study-tracks` | 200; active tracks only, names in the request language |
| MD-02 | `GET /study-tracks/{id}/subjects` | 200; active subjects of that track only |
| MD-03 | Subjects of an unknown / inactive track (deactivate one in the admin panel first) | 404 `STUDY_TRACK_NOT_FOUND` |
| MD-04 | `GET /academic-years` | 200; newest first, with `start_date` / `end_date` |
| MD-05 | No token | 401 `UNAUTHENTICATED` |
| MD-06 | `Accept-Language: ar` vs `en` | track / subject names switch language |

## 4. Test cases: create

As Mona with her own context unless stated.

| ID | Case | Body / steps | Expected |
|---|---|---|---|
| CR-01 | Valid course | all fields, `term: first` | **201**; `code` like `CO-123456`; listed afterwards |
| CR-02 | Term `both` | `term: both` | 201; `term: "both"` |
| CR-03 | With image | multipart, a jpg ≤ 2 MB | 201; `image` is a URL |
| CR-04 | Image too big / wrong type | 3 MB jpg; a pdf | 422 `VALIDATION_ERROR` on `image` |
| CR-05 | Name rules | `""`, `"A"`, 256 characters, `"  Math  "` | 422 for the first three; the last is saved trimmed |
| CR-06 | Missing fields | omit each required field in turn | 422, field error |
| CR-07 | Bad term | `term: third` | 422 `VALIDATION_ERROR` (`term`) |
| CR-08 | Subject of another track | track A + subject of track B | 422 `SUBJECT_NOT_IN_STUDY_TRACK` |
| CR-09 | Inactive track / subject | deactivate them in the admin panel, then create | 422 `STUDY_TRACK_INACTIVE` / `SUBJECT_INACTIVE` |
| CR-10 | Unknown ids | `study_track_id: 999999`, `subject_id: 999999`, `academic_year_id: 999999` | 422 `STUDY_TRACK_NOT_FOUND` / `SUBJECT_NOT_FOUND` / `ACADEMIC_YEAR_NOT_FOUND` |
| CR-11 | Ignored fields | send `code`, `teacher_user_role_id` (Youssef), `id` | course created for **Mona** with a generated code |
| CR-12 | Secretary creates | Ahmed with `X-Teacher-Id` = Mona | 201; the course is **Mona's** (visible in Mona's list) |
| CR-13 | Suspended / expired secretary | Ahmed → Youssef; Nour → Youssef | 403 `SECRETARY_RELATION_INACTIVE` / `SECRETARY_RELATION_EXPIRED` |
| CR-14 | Inactive teacher | Ahmed with his **own** teacher id | 403 `TEACHER_ROLE_NOT_ACTIVE` |
| CR-15 | Rate limit | more than 60 writes within a minute | 429 `TOO_MANY_REQUESTS` with `Retry-After` |

## 5. Test cases: list & details

| ID | Case | Expected |
|---|---|---|
| LS-01 | `GET /teacher/courses` as Mona | only Mona's 3 courses (plus any you created), newest first, each with `students_count` |
| LS-02 | Same as Ahmed (Mona's context) | the same list |
| LS-03 | Filters | `?term=first`, `?academic_year_id=`, `?study_track_id=`, `?subject_id=` return only matching courses |
| LS-04 | Search | part of a name; the course code | matching courses only; `%` / `_` in the search are treated literally |
| LS-05 | Pagination | `?per_page=1&page=2`; `?per_page=101` | 1 item; 422 |
| LS-06 | Details | `GET /teacher/courses/{id}` | 200, full object |
| LS-07 | Another teacher's course | Mona opens a Youssef course id | 403 `COURSE_NOT_ACCESSIBLE` |
| LS-08 | Unknown / deleted course | random UUID; a deleted course | 404 `COURSE_NOT_FOUND` |
| LS-09 | Bad id | `/teacher/courses/abc` | 404 (route) |
| LS-10 | `students_count` | enroll / suspend / expel a student (Feature 6) | counts active + suspended only |

## 6. Test cases: edit & delete

| ID | Case | Expected |
|---|---|---|
| UP-01 | Rename | `PATCH {name}` → 200; other fields unchanged |
| UP-02 | Change term / year | 200 |
| UP-03 | Change track without a matching subject | `PATCH {study_track_id: other}` keeping an old subject → 422 `SUBJECT_NOT_IN_STUDY_TRACK` |
| UP-04 | Image | `POST` multipart + `_method=PATCH` with a new image → 200, new URL |
| UP-05 | Code is read-only | `PATCH {code: "CO-000001"}` → 200, code unchanged |
| UP-06 | Another teacher's course | 403 `COURSE_NOT_ACCESSIBLE`; nothing changes |
| DL-01 | Delete | `DELETE` → 200 `data: null`; gone from the list; `GET` → 404 `COURSE_NOT_FOUND` |
| DL-02 | Delete twice | 404 `COURSE_NOT_FOUND` |
| DL-03 | Delete another teacher's course | 403 `COURSE_NOT_ACCESSIBLE` |
| DL-04 | Groups / enrollments after delete | Feature 4 / 6 calls on that course → 404 `COURSE_NOT_FOUND` |

## 7. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Academic data | create / edit / deactivate a study track, subject and academic year; deactivated items disappear from MD-01 / MD-02 and are rejected on create (CR-09) |
| AD-02 | Delete used master data | delete a subject used by a course | refused: "This record is still in use" |
| AD-03 | Teaching → Courses | every teacher's courses, filters by instructor / track / year |
| AD-04 | New course for a teacher | *New course* → pick Youssef → fill in | created; visible in Youssef's API list; audit log actor = the admin |
| AD-05 | Edit / delete from the panel | same rules as UP / DL (e.g. a subject of another track shows the `SUBJECT_NOT_IN_STUDY_TRACK` notification) |
| AD-06 | Support admin | read-only: no *New course*, *Edit* or *Delete* |

## 8. Mobile checks

| ID | Check |
|---|---|
| MB-01 | The subject picker reloads, and the selection resets, when the track changes. |
| MB-02 | `both` is selectable as a term and displays correctly. |
| MB-03 | Image upload on edit works and shows the new image. |
| MB-04 | After a delete on another device, opening the course shows a friendly "not found" and returns to the list. |
| MB-05 | Secretary mode shows the picked teacher's courses only, and switching teachers refreshes the list. |
| MB-06 | Arabic / English: names and messages follow the app language. |

## 9. Bug report

Include the account, `X-Teacher-Id`, the request (method, URL, body), the response (status, `error_code`, `message`, `errors`) and the test case ID. Never include tokens or passwords.
