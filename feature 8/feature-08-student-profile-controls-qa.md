# Feature 08 — Student profile & controls · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-08-student-profile-controls.md](../features/feature-08-student-profile-controls.md) · mobile guide: [feature-08-student-profile-controls-mobile.md](feature-08-student-profile-controls-mobile.md)

---

## 1. Scope

* Student profile reached through an enrollment: personal data, activity, devices, enrollment, permission states.
* Device tracking on **every** authenticated request (`X-Device-*` headers), last login / last seen.
* Per-enrollment permissions (board, private messages) with inheritance.
* Student Excel and attendance exports through temporary links.
* Admin panel: *Enrollments → view* (Profile & Device section, Edit Permissions).

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

Password `Darsly#User2026`; admin `admin@darsly.test` / `Darsly#Admin2026` at `/admin`.

| Seeded student | Situation |
|---|---|
| Omar (math with Mona) | one current Android device; explicit permissions: board **off**, messages **on** |
| Sara | switched from an iPhone to a Samsung yesterday (two devices) |
| Laila | no recorded login |
| others | inherit the course defaults |

Get enrollment ids from `GET /teacher/course-enrollments?course_id=…` (Mona's context). Postman: `postman/Darsly-Feature-08-Student-Profile-Controls.postman_collection.json` (folder 08 restores the seeded state).

---

## 3. Test cases: profile

| ID | Case | Expected |
|---|---|---|
| PF-01 | Mona opens Omar's math enrollment | 200, all sections; `counts.transactions` = his transaction count |
| PF-02 | Ahmed (Mona's secretary) | the same 200 |
| PF-03 | Youssef (another teacher) opens it | **404** `ENROLLMENT_NOT_FOUND` (not 403) |
| PF-04 | Unknown / deleted enrollment; enrollment of a deleted course | 404 `ENROLLMENT_NOT_FOUND` |
| PF-05 | A secretary not linked to the teacher | 403 `SECRETARY_NOT_LINKED_TO_TEACHER` |
| PF-06 | Suspended / expelled enrollment | 200 (still readable) |
| PF-07 | Laila | `last_login_at: null`, `no_recorded_login: true` |
| PF-08 | Sara | `current_device` Samsung, `previous_device` iPhone, `device_changed_at` set |
| PF-09 | Secrets | no device id (raw or hashed), no token, no other roles of the student in the payload |
| PF-10 | `PATCH …/profile` | 405 (read-only) |
| PF-11 | `Accept-Language: en` | governorate and messages in English |

## 4. Test cases: device tracking

| ID | Case | Expected |
|---|---|---|
| DV-01 | Call any authenticated endpoint as Omar with a new `X-Device-Id` | his profile shows the new current device; the old one becomes previous |
| DV-02 | Same device again within 5 minutes | no new write (last seen unchanged until the window passes) |
| DV-03 | Switch back to the old device id | the existing row is reused (no duplicate) and becomes current at once |
| DV-04 | Invalid id (`ab`, spaces, 200 chars) | request succeeds; no device created |
| DV-05 | Invalid metadata (huge model name) | request succeeds; value truncated |
| DV-06 | Public endpoint / wrong token | nothing tracked |
| DV-07 | Login | `last_login_at` updated; a failed login doesn't change it |

## 5. Test cases: permissions

| ID | Case | Body | Expected |
|---|---|---|---|
| PM-01 | Read Omar's | — | board `{stored: false, effective: false, inherited: false}`, messages `{stored: true, effective: true}` |
| PM-02 | Reset board to default | `{can_post_in_board: null}` | `inherited: true`, `effective` = course default |
| PM-03 | Omitted key | `{can_send_private_messages: false}` | board unchanged |
| PM-04 | No-op (same values) | — | 200, no audit row |
| PM-05 | Empty body | `{}` | 422 (`permissions`) |
| PM-06 | Unknown key | `{can_upload_files: true}` | 422 on that key |
| PM-07 | Non-boolean | `"true"`, `1`, `[true]` | 422 |
| PM-08 | Secretary changes it | | 200; audit `enrollment.permissions_updated` with the secretary as author |
| PM-09 | Another teacher | | 404 |
| PM-10 | Course messaging default (Feature 21) | turn the course's private messages on, then read an inheriting student | `can_send_private_messages.effective: true` |

## 6. Test cases: exports

| ID | Case | Expected |
|---|---|---|
| EX-01 | Student Excel link | 200 `{type: student_xlsx, url, expires_at}`; the URL has no ids and no query string |
| EX-02 | Open the URL without any header | downloads `student-{code}-{course}-{date}.xlsx`; 3 RTL sheets; payments sheet has only `enrollment` / `renewal` rows |
| EX-03 | Open it again within 10 minutes | downloads again (audited each time) |
| EX-04 | After 10 minutes / a mistyped token | 410 `EXPORT_LINK_EXPIRED` |
| EX-05 | Mona blocked (admin) after the link was issued | opening → 403 `LOGIN_NOT_ALLOWED` |
| EX-06 | Formula injection | a payment message `=1+1` shows as text in Excel |
| EX-07 | Attendance link + download | CSV with BOM, columns `date,item,type,group,status,views,score,percentage`, one row per assigned lecture / exam |
| EX-08 | Another teacher's enrollment | 404, no link |

## 7. Test cases: status (Feature 6 reuse)

| ID | Case | Expected |
|---|---|---|
| ST-01 | Suspend from the profile | Feature 6 PATCH 200; profile shows `status: suspended`, `has_access: false` |
| ST-02 | Reactivate | back to active |
| ST-03 | "+30 days" | the app sends a computed `subscription_ends_at`; profile shows the new end date |

## 8. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Enrollments → view Sara's | Profile & Device section with both devices and the change time; no device ids |
| AD-02 | Edit Permissions (Inherit / Allowed / Not allowed) | saved; audit actor = the admin; `updated_by_admin_id` set |
| AD-03 | Support admin | section visible, no Edit Permissions button |

## 9. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Device headers are sent on every call (check in a proxy). |
| MB-02 | Permission switches show the inherited state and can reset to default. |
| MB-03 | Export opens in the browser / share sheet without a login prompt. |
| MB-04 | "No recorded login" wording for Laila. |

## 10. Bug report

Account, `X-Teacher-Id`, enrollment id, request and full response (`error_code`, `errors`), the device headers sent, the test ID and the time.
