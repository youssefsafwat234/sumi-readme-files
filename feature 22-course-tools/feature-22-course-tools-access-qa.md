# Feature 22 — Course tools & access window · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-22-course-tools-access.md](../features/feature-22-course-tools-access.md) · mobile guide: [feature-22-course-tools-access-mobile.md](feature-22-course-tools-access-mobile.md)

---

## 1. Scope

* Tools summary (one request, every tool, null for tools not built).
* Live stream: https rule, provider detection, on / off stamps, student visibility.
* Access window: validation, states, enforcement on every student content endpoint, staff unaffected, attempts closed.
* Student course page.
* Admin panel: Courses → view → «Live stream» and «Access window» actions.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

Demo course: live stream **off** (YouTube link, last ended yesterday); access window opens **10/10/2026 17:00 (Cairo)**, no close; 12 notifications, 7 categories, 128 questions.

Postman: `postman/Darsly-Feature-22-Course-Tools-Access.postman_collection.json` (restores the seeded state).

---

## 3. Test cases: tools summary

| ID | Case | Expected |
|---|---|---|
| TS-01 | Mona → `GET /tools` | `notifications.count: 12`, `categories.count: 7` + 3 names, `question_bank.count: 128` + by type, `access_window.state: open`, `meeting: null`, `discussion_board: null` |
| TS-02 | Ahmed (secretary) | 200 |
| TS-03 | Youssef | 403 `COURSE_NOT_ACCESSIBLE` |
| TS-04 | A new empty course | zeros, `live_stream.enabled: false` |

## 4. Test cases: live stream

| ID | Case | Expected |
|---|---|---|
| LS-01 | `enabled: true` without a link | 422 `LIVE_STREAM_URL_REQUIRED` |
| LS-02 | `http://…`, `ftp://…`, `not a link` | 422 `INVALID_LIVE_STREAM_URL` |
| LS-03 | YouTube / Facebook / Zoom / Meet / Vimeo link | `provider` youtube / facebook / zoom / google_meet / other |
| LS-04 | Turn on | `started_at` = now |
| LS-05 | Turn off | `last_ended_at` = now |
| LS-06 | Remove the link while on | 422 `LIVE_STREAM_URL_REQUIRED` |
| LS-07 | Student page while off / on | `live_stream: null` / `{url, title, provider}` |
| LS-08 | Student with an expired subscription / window closed | `live_stream: null` |

## 5. Test cases: access window

| ID | Case | Expected |
|---|---|---|
| AW-01 | `closes_at` before `opens_at` | 422 `INVALID_ACCESS_WINDOW` |
| AW-02 | `opens_at` tomorrow | `state: not_yet_open` |
| AW-03 | `opens_at: null` | the bound cleared; `open` |
| AW-04 | Student, window not open: lectures list, lecture, start a view, files list, file access, exams list, start an attempt | every one 403 `COURSE_NOT_OPEN_YET` with `details` |
| AW-05 | Window closed: start an attempt | 403 `COURSE_ACCESS_CLOSED` |
| AW-06 | Student course page while closed | 200 with `access.state: closed` |
| AW-07 | Teacher / secretary screens while closed | unaffected |
| AW-08 | Attempt started, window closes in 10 min, exam duration 60 | `deadline_at` = the close; after it the attempt is submitted with `exam_closed` |
| AW-09 | Audit | `course.access_window_updated` / `course.live_stream_updated` with before / after |

## 5.1 Test cases: save all tools (`PATCH /tools`)

| ID | Case | Expected |
|---|---|---|
| AT-01 | All three blocks valid | 200, the summary shows the new stream, window and messaging settings; three audit rows |
| AT-02 | Valid stream + window closing before it opens | 422 `INVALID_ACCESS_WINDOW`, `errors[0].key = access_window.closes_at`; the stream is **not** changed |
| AT-03 | `live_stream: {enabled: true}` with no link saved | 422 `LIVE_STREAM_URL_REQUIRED` on `live_stream.url` |
| AT-04 | Empty body `{}` | 422 `VALIDATION_ERROR` on `live_stream`, `access_window`, `private_messages` |
| AT-05 | `private_messages.allow_images: "maybe"` | 422 on `private_messages.allow_images` |
| AT-06 | Another teacher's course | 403 `COURSE_NOT_ACCESSIBLE` |

## 6. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Course → Live stream action | same rules |
| AD-02 | Course → Access window action | same rules |

## 7. Mobile checks

| ID | Check |
|---|---|
| MB-01 | «الكورس هيفتح يوم …» shown before the window. |
| MB-02 | Live button appears only when the stream is on. |
| MB-03 | Content screens show the window message instead of a generic error. |

## 8. Bug report

Course id, window values (with offsets), request and full response (`error_code`, `details`), the test ID and the time (and timezone).
