# Feature 19 — Student progress & attendance · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-19-progress-attendance.md](../features/feature-19-progress-attendance.md) · mobile guide: [feature-19-progress-attendance-mobile.md](feature-19-progress-attendance-mobile.md)

---

## 1. Scope

* Lecture statuses, exam statuses, the three percentages, `null` when nothing is assigned.
* Assigned items (published, out, the student's groups incl. past memberships).
* Recalculation triggers, debounce, stale flag, nightly command.
* Teacher views (student, group roster, lecture), student view, attendance CSV.
* Admin panel: Enrollments → view (Progress section, Recalculate).

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
php artisan queue:work          # recalculation jobs (30 s debounce)
```

| Student | Seeded |
|---|---|
| Omar Khaled | Chapter 1 · Part 1 **viewed twice** (unit 1 exam 14 / 20, passed); Part 2 **viewed once**; Production practical **not completed**; Chapter 2 · Part 1 **not started** |
| Nour Elsayed (`nour.elsayed@darsly.test`) | new in group A, nothing done |

Postman: `postman/Darsly-Feature-19-Progress-Attendance.postman_collection.json`.

---

## 3. Test cases: statuses & numbers

| ID | Case | Expected |
|---|---|---|
| PG-01 | Omar's progress | statuses as seeded; `related_exam` on Part 1 = 14 / 20 passed |
| PG-02 | Percentages | lectures = completed / assigned; attendance = (viewed lectures + attempted exams) / (lectures + exams); course = (completed + passed) / (lectures + exams) |
| PG-03 | Nour Elsayed | all items `not_started` / `not_attempted`; percentages 0 |
| PG-04 | A student in a group with nothing published | percentages `null` |
| PG-05 | Teacher marks a lecture watched (no view) | `viewed_once`; counts in attendance |
| PG-06 | Scheduled lecture / exam not open yet / draft | not counted |
| PG-07 | Item of a group the student isn't in | not counted |
| PG-08 | Remove the student from a group | items already out stay; new ones of that group don't count |
| PG-09 | Exam attempted, failed | attendance counts it; course progress doesn't |

## 4. Test cases: recalculation

| ID | Case | Expected |
|---|---|---|
| PR-01 | Start a lecture view | `stale: true` → after ≤ 30 s fresh numbers |
| PR-02 | Reach the watched threshold | status `viewed_once` |
| PR-03 | Submit an exam | exam item `attempted`, percentages updated |
| PR-04 | Publish a new lecture to the group | assigned count +1 for every student of the course |
| PR-05 | 10 triggers in a few seconds | one job per enrollment |
| PR-06 | `php artisan progress:recalculate --enrollment={id}` | that enrollment recalculated |
| PR-07 | `POST …/progress/recalculate` | fresh numbers immediately |

## 5. Test cases: teacher & student reads

| ID | Case | Expected |
|---|---|---|
| PV-01 | Roster `sort=course_progress&direction=asc` | lowest first |
| PV-02 | `below=50` | under 50 % or not calculated |
| PV-03 | Lecture progress `?status=viewed_twice` | Omar; `counts` per status |
| PV-04 | Student `GET /student/courses/{math}/progress` | own numbers |
| PV-05 | Student without access (suspended) | 404 `PROGRESS_NOT_AVAILABLE` |
| PV-06 | Not enrolled | 404 `COURSE_NOT_FOUND` |
| PV-07 | Another teacher's enrollment | 404 / 403 |
| PV-08 | Attendance CSV (Feature 8 link) | `date,item,type,group,status,views,score,percentage`, one row per item |

## 6. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Enrollment view → Progress & attendance | the 3 percentages, counts, last activity, calculated at |
| AD-02 | Recalculate progress | numbers refreshed |

## 7. Mobile checks

| ID | Check |
|---|---|
| MB-01 | «لسه مفيش محتوى» for `null`. |
| MB-02 | Status labels and colours per lecture. |
| MB-03 | Related exam result next to the lecture. |

## 8. Bug report

Enrollment / course / group / lecture ids, the response (`summary`, item statuses, `stale`, `calculated_at`), what the student did and when, the test ID.
