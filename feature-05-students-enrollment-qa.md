# Feature 05 — Students & Enrollment · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-06-enrollment-transactions.md](../features/feature-06-enrollment-transactions.md) (the old Feature 05 API was replaced by Feature 06) · mobile guide: [feature-05-students-enrollment-mobile.md](feature-05-students-enrollment-mobile.md) · related: [Feature 06 QA](feature-06-enrollment-transactions-qa.md), [Feature 07 QA](feature-07-group-student-management-qa.md)

---

## 1. Scope

* Student lookup by exact code / email before enrolling (also the QR flow).
* Enroll a student into a course and its groups (new and restored), suggested amount.
* Course student list: filters, search, `expiry` filter, pagination.
* Enrollment details: `has_access`, `access_reason`, payment summary (detailed in Feature 06).
* Edit status, dates and groups; automatic status rules; remove from the course.
* `GET /teacher/settings` (automatic transactions flag).
* Student app: own enrollments, read-only, nothing financial.
* Access: the course's teacher or a validly linked secretary only.

Not here: payments, renewals and history (Feature 06 QA); the group roster and bulk actions (Feature 07 QA).

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

Password `Darsly#User2026`; admin `admin@darsly.test` at `/admin`.

| Account | Use |
|---|---|
| `mona.mahmoud@darsly.test` | Teacher A (automatic transactions **ON**) |
| `youssef.ibrahim@darsly.test` | Teacher B (automatic transactions **OFF**) |
| `ahmed.hassan@darsly.test` | Mona's secretary (active link); suspended link with Youssef; also a student |
| `nour.salem@darsly.test` | Mona's secretary; **expired** link with Youssef; also a student |
| `laila.fathy@darsly.test` | inactive secretary role; a student |

Seeded enrollments (Mona's Mathematics course "مراجعة الرياضيات - الصف الثالث الثانوي" unless stated):

| Student | Groups | Status | Dates | Paid |
|---|---|---|---|---|
| Omar Khaled | A | active | 2026-10-01 → 2026-11-01 | 300.00 + renewal 300.00 |
| Sara Ali | B | suspended | none | not specified |
| Laila Fathy | evening (free) | active | none | 0.00 |
| Nour Salem | A + evening | active | none | 300.00 |
| Omar (Physics) | مجموعة الجمعة | active | none | 850.00 |
| Ahmed (Physics) | مجموعة الجمعة | expelled | none | 850.00 |
| Sara (Physics) | — | **deleted** | — | 850.00 |
| Nour, Omar (Youssef's English course) | … | active / suspended | none | 200.00 / 0.00 |

> Omar's dates are fixed: after 2026-11-01 he shows `access_reason: expired`. Use the PATCH cases below to set dates relative to today.

Get ids from `GET /teacher/courses` and `GET /teacher/courses/{id}/groups`; student codes from `GET /teacher/course-enrollments`. Postman: `postman/Darsly-Feature-06-Enrollments-Transactions.postman_collection.json` (folders 02, 03, 04, 05, 08, 09, 12).

---

## 3. Test cases: student lookup

As Mona on Mathematics.

| ID | Case | Request | Expected |
|---|---|---|---|
| SL-01 | Not enrolled | `?course_id=…&code=<Ahmed's student code>` | 200; name, image, government; `enrollment: null`, `previously_enrolled: false` |
| SL-02 | Already enrolled | Omar's code | 200; `enrollment` = his enrollment (with `access_reason`) |
| SL-03 | Enrolled before | Sara's code on **Physics** | `enrollment: null`, `previously_enrolled: true` |
| SL-04 | Email | `&email= OMAR.KHALED@darsly.test ` (spaces, capitals) | 200, Omar |
| SL-05 | Unknown / teacher / secretary code | `99999999`; Mona's teacher code; a secretary code | 404 `STUDENT_NOT_FOUND` (same answer for all) |
| SL-06 | Inactive student | a student whose student role is inactive | 404 `STUDENT_NOT_FOUND` |
| SL-07 | Identifier rules | neither; both `code` and `email`; `code=abc`; `name=Omar` | 422 `VALIDATION_ERROR` |
| SL-08 | Other enrollments ignored | Nour's code on Physics (she is only in Math / English) | `enrollment: null` |
| SL-09 | Another teacher's course | Youssef on Mona's course | 403 `COURSE_NOT_ACCESSIBLE` |
| SL-10 | Secretary | Ahmed in Mona's context | 200 |
| SL-11 | QR | scan a student's QR in the app | the QR holds the role **code**; the same result as typing it |
| SL-12 | Throttle | 31 lookups / creates within a minute | the 31st → 429 `TOO_MANY_REQUESTS` |

## 4. Test cases: enroll

| ID | Case | Body | Expected |
|---|---|---|---|
| EN-01 | By code, two groups | `student_code`, A + evening, `transaction: {amount: 350, payment_method: "cash"}` | **201**; active; both groups; `enrollment` transaction 350.00 (+ `group_added` ×2 for Mona) |
| EN-02 | By role id | `student_user_role_id` from the lookup | 201 |
| EN-03 | Suggested amount | `GET …/suggested-amount?course_id=&group_ids[]=A&group_ids[]=evening` | `"300.00"`, `complete: true`; with group B → `amount: null`, `complete: false`, `partial_amount` |
| EN-04 | Amount rules | `0`; `null`; omitted; `-1`; `10.555` | `"0.00"`; `null`; `null`; 422; 422 |
| EN-05 | Payment method | `bitcoin` | 422 (`transaction.payment_method`) |
| EN-06 | Groups | none; duplicate ids; a Physics group; an inactive group; a full group | 422; 422; 422 `INVALID_GROUP_ASSIGNMENT`; 422 `GROUP_INACTIVE`; 409 `GROUP_CAPACITY_REACHED` |
| EN-07 | Status | `suspended`; `expelled` | created with that status |
| EN-08 | Dates | start = end; start after end | 422 `INVALID_SUBSCRIPTION_DATES` |
| EN-09 | Both identifiers | `student_code` + `student_user_role_id` | 422 |
| EN-10 | Already enrolled | Omar again | 409 `STUDENT_ALREADY_ENROLLED` |
| EN-11 | Restore | enroll Sara in Physics | **200**, same id as before, `restored: true`; old groups **not** back; omitted dates are `null`; old transactions still in the history |
| EN-12 | Transfer images | multipart with `transaction_images[]` | 201; images listed on the transaction (Feature 06) |
| EN-13 | Secretary | Ahmed enrolls for Mona | 201; transactions `created_by.role: "secretary"` |
| EN-14 | Another teacher's course | Youssef → Mona's course | 403 `COURSE_NOT_ACCESSIBLE` |

## 5. Test cases: list & details

| ID | Case | Expected |
|---|---|---|
| LS-01 | List as Mona | only Mona's courses' enrollments; newest first |
| LS-02 | `?course_id=` / `?group_id=` / `?status=suspended` | filtered correctly |
| LS-03 | `?search=` part of a name / code / email / phone | matches; `%` and `_` are literal |
| LS-04 | `?course_id=` of Youssef's course | 403 `COURSE_NOT_ACCESSIBLE` |
| LS-05 | Expiring | set Omar's end to today + 3 days; `?expiry=expiring` → Omar listed; `&expiring_within_days=2` → not listed |
| LS-06 | Expired | set an end date to yesterday; `?expiry=expired` → listed, `has_access: false`, `access_reason: "expired"` |
| LS-07 | No end date | Laila (no dates) is in neither `expiring` nor `expired` |
| LS-08 | Expiry validation | `expiry=soon`; `expiring_within_days=3` alone; `expiry=expired&expiring_within_days=3`; `0`; `366` | 422 |
| LS-09 | Details | `GET …/{id}` → every field of the enrollment object incl. `access_reason`, `last_payment`, `total_paid`, `currency` |
| LS-10 | Payment summary (seed) | Omar: `total_paid "600.00"`, `last_payment.amount "300.00"`; Sara: `"0.00"` / `null`; Laila (paid 0.00): `"0.00"` / `null` |
| LS-11 | No private data | the response never contains transaction messages, transfer images or passwords |
| LS-12 | Another teacher's enrollment | 403 `ENROLLMENT_NOT_ACCESSIBLE`; unknown / deleted → 404 `ENROLLMENT_NOT_FOUND` |

## 6. Test cases: access reason

| ID | Setup | Expected `has_access` / `access_reason` |
|---|---|---|
| AR-01 | Nour (active, active groups, no dates) | `true` / `null` |
| AR-02 | Sara (suspended) | `false` / `suspended` |
| AR-03 | Ahmed in Physics (expelled) | `false` / `expelled` |
| AR-04 | Deactivate every group of an active student (Feature 4) | `false` / `no_active_group` |
| AR-05 | `PATCH {subscription_started_at: <tomorrow>}` | `false` / `not_started` |
| AR-06 | `PATCH {subscription_ends_at: <now>}` | `false` / `expired` (the end is exclusive) |
| AR-07 | Suspended **and** expired | `suspended` (status is reported first) |
| AR-08 | Student view | the same `access_reason` in `GET /student/course-enrollments/{id}` |

## 7. Test cases: edit status, dates, groups

| ID | Case | Body | Expected |
|---|---|---|---|
| UP-01 | Suspend | `{status: "suspended"}` | suspended; `status_changed` (Mona: ON) |
| UP-02 | Add a group to a suspended student | `{groups: {attach: [evening]}}` | re-activated automatically |
| UP-03 | Remove the last active group | `{groups: {detach: [A]}}` on a student with only A | suspended automatically |
| UP-04 | Expelled stays | add a group to an expelled student | stays `expelled` |
| UP-05 | Active without an active group | `{status: "active"}` with no active group | 422 `ACTIVE_ENROLLMENT_REQUIRES_ACTIVE_GROUP`; nothing saved |
| UP-06 | Same group in attach and detach | | 422 (`groups`) |
| UP-07 | Group of another course | | 422 `INVALID_GROUP_ASSIGNMENT` |
| UP-08 | +30 days | the app sends the new `subscription_ends_at` | dates updated; a `renewal` without amount with old / new dates |
| UP-09 | Clear a date | `{subscription_ends_at: null}` | `null`; renewal records the change |
| UP-10 | Omit vs null | `{status: "active"}` only | dates unchanged |
| UP-11 | No-op | send the current values | 200; no new transaction |
| UP-12 | Youssef (OFF) | status / group changes | applied, but **no** `status_changed` / `group_*` rows; date changes still recorded |

## 8. Test cases: remove & teacher settings

| ID | Case | Expected |
|---|---|---|
| RM-01 | Delete | 200 `data: null`; gone from lists; `GET` → 404; history still readable (Feature 06) |
| RM-02 | Delete twice | 404 `ENROLLMENT_NOT_FOUND` |
| RM-03 | Enroll again | restored (EN-11) |
| ST-01 | `GET /teacher/settings` as Mona / Ahmed (Mona's context) | `automatic_transactions: true` |
| ST-02 | As Youssef | `false` |
| ST-03 | Youssef with Mona's `X-Teacher-Id`; an unlinked secretary | 403 |
| ST-04 | `PATCH /teacher/settings` | 405 (read-only) |

## 9. Test cases: student app

| ID | Case | Expected |
|---|---|---|
| SA-01 | Nour logs in → `GET /student/course-enrollments` | her Math and English enrollments; no `total_paid`, `last_payment`, `amount` or history |
| SA-02 | One of hers | 200 with `has_access`, `access_reason` |
| SA-03 | Somebody else's / a deleted one | 404 `ENROLLMENT_NOT_FOUND` |
| SA-04 | Teacher routes with a student only | 403 / context errors |

## 10. Teacher context & security

| ID | Case | Expected |
|---|---|---|
| SC-01 | Missing / malformed `X-Teacher-Id` | 422 `TEACHER_CONTEXT_HEADER_REQUIRED` / `INVALID_TEACHER_CONTEXT` |
| SC-02 | Ahmed in Youssef's context (suspended link) | 403 `SECRETARY_RELATION_INACTIVE` |
| SC-03 | Nour in Youssef's context (expired link) | 403 `SECRETARY_RELATION_EXPIRED` |
| SC-04 | Laila (inactive secretary) | 403 |
| SC-05 | Unauthenticated | 401 |
| SC-06 | Arabic / English | same `error_code`, translated `message` |

## 11. Admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | *Teaching → Enrollments* | every teacher's enrollments; create / edit / delete on the teacher's behalf |
| AD-02 | Same rules | inactive group, full group, active without a group → the same errors as the API |
| AD-03 | Automatic transactions | *Users → Mona → Disable automatic transactions* → `GET /teacher/settings` returns `false` on the next request |
| AD-04 | Payment method | the admin forms don't offer `payment_method` yet (stored as `null`) — known limitation |

## 12. Bug report

Account, `X-Teacher-Id`, course / enrollment ids, the request and full response (`error_code`, `errors`), the test ID, and the time (dates matter). No tokens or passwords.
