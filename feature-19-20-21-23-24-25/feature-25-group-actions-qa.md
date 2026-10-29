# Feature 25 — Group-wide actions · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-25-group-actions.md](../features/feature-25-group-actions.md) · mobile guide: [feature-25-group-actions-mobile.md](feature-25-group-actions-mobile.md)

---

## 1. Scope

* The group actions menu (L25.9) and every action on the **whole** group: add to another group, remove all from this group, sort, students / attendance Excel export, activate until a date, expel inactive.
* The preview → `expected_count` confirmation and `GROUP_CHANGED`.
* Not in scope: actions on selected students (Feature 07, unchanged), sorting by attendance, multi-group exports, notifications.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

| Data | Seeded |
|---|---|
| «مجموعة السبت» (Mona, «مراجعة الرياضيات - الصف الثالث الثانوي», max 50) | 42 students `saturday.student01…42@darsly.test` |
| Active | 38 — of which كريم مصطفى (ended 1 Oct) and حبيبة رضا (ended 20 Sep) |
| Suspended | 3 — سارة محمود, يوسف هاني, مريم عادل |
| Expelled | 1 — أحمد سمير |
| Other groups of the course | المجموعة A (max 50), المجموعة B (max 30) |

Teacher `mona.mahmoud@darsly.test`, linked secretary `ahmed.hassan@darsly.test`, other teacher `youssef.ibrahim@darsly.test`; password `Darsly#User2026`. Postman: `postman/Darsly-Feature-25-Group-Actions.postman_collection.json` (folder 10 restores the group).

«غير مفعّل» = suspended, or active whose subscription end is now or earlier. Expected at seed time: **5**.

---

## 3. Test cases: confirmation & safety

| ID | Case | Expected |
|---|---|---|
| GA-01 | Any write with `enrollment_ids` (or `course_enrollment_ids`) in the body | 422 `VALIDATION_ERROR` on that key, nothing changes |
| GA-02 | Write with an old `expected_count` (add / remove a student in between) | 409 `GROUP_CHANGED`, `details` has the new counts, nothing changes |
| GA-03 | Two devices confirm the same action at the same moment | one succeeds, the other gets `GROUP_CHANGED` (or a no-op result), never a half-done group |
| GA-04 | `group_action_max` lowered to 1 (config), group of 2 | 422 `GROUP_ACTION_LIMIT_EXCEEDED` `{limit, count}` |
| GA-05 | Missing `expected_count` | 422 `VALIDATION_ERROR` |

## 4. Test cases: add to another group / remove all

| ID | Case | Expected |
|---|---|---|
| AG-01 | Preview «مجموعة السبت» → المجموعة B | `count 41`, `to_add 41`, `skipped_expelled 1`, `available_seats` < 41 |
| AG-02 | Confirm → المجموعة B | 422 `GROUP_CAPACITY_EXCEEDED` `{available, requested: 41}`, nobody added |
| AG-03 | Confirm → an empty active group without limit | `{added: 41, skipped: 0}`; students keep «مجموعة السبت»; statuses unchanged; أحمد سمير not added |
| AG-04 | Same again | `{added: 0, skipped: 41}`, no new history rows |
| AG-05 | Target = same group / inactive group / group of another course | 422 `INVALID_TARGET_GROUP` |
| RA-01 | Remove-all preview on a group whose students have no other active group | `will_be_suspended` = number of active students |
| RA-02 | Confirm | `{removed, suspended}` = preview; memberships of other groups kept; payments kept; `group_removed` + `status_changed` in each student's history |

## 5. Test cases: sort & exports

| ID | Case | Expected |
|---|---|---|
| SO-01 | `sort=name` / `code` | A→Z / ascending code |
| SO-02 | `sort=ends_at` | ended students first, students without an end date last |
| SO-03 | `sort=joined_at` | most recently added to the group first |
| SO-04 | `sort=last_seen_at` | never-seen students first, then oldest activity |
| SO-05 | `sort=attendance` | 422 on `sort` |
| EX-01 | Excel | file `group-…-students-YYYY-MM-DD.xlsx`, `Cache-Control: private, no-store`, sheets «الطلاب» (42 rows + header, every status) and «الاشتراكات», right-to-left, Arial |
| EX-02 | Excel with `status=suspended&sort=name` | only the 3 suspended students, by name |
| EX-03 | Attendance Excel (`attendance.xlsx`, `Accept-Language: ar`) | file `group-…-attendance-YYYY-MM-DD.xlsx`; one RTL sheet «الحضور», Arial, same header style as the students export; Arabic headers (كود الطالب … النسبة) and labels (المحاضرة / الامتحان, لسه, شافها مرة …); each student's rows equal that student's Feature 19 CSV |
| EX-04 | Exports by the secretary | allowed; audit `group.exported` shows the secretary |

## 6. Test cases: activate & expel inactive

| ID | Case | Expected |
|---|---|---|
| AC-01 | Preview `active_only` / `active_and_suspended` | 38 / 41, `skipped_expelled 1` |
| AC-02 | `ends_at` in the past / `scope=everyone` | 422 on `ends_at` / `scope` |
| AC-03 | Confirm `active_only` with a future date | `updated` + `unchanged` = 38; suspended and expelled untouched; each changed student has a `renewal` **without amount**; no new payment |
| AC-04 | Same date again | `{updated: 0, unchanged: 38}` |
| AC-05 | `active_and_suspended` | suspended students become active (`status_changed`), expelled untouched |
| AC-06 | A student whose subscription starts after the date | 422 `INVALID_SUBSCRIPTION_DATES` with `details.enrollment_id`, nobody changed |
| IN-01 | Inactive list at seed | 5: three `suspended`, كريم مصطفى and حبيبة رضا `expired` with their dates |
| IN-02 | A student whose end is exactly now | listed as `expired` |
| IN-03 | Expel with `expected_count` = count | `{expelled: n}`; each is `expelled`, out of this group, other groups and payments kept |
| IN-04 | Inactive list again | `count 0` |

## 7. Test cases: authorization, localization, admin

| ID | Case | Expected |
|---|---|---|
| AU-01 | Linked secretary | every action allowed |
| AU-02 | Other teacher (Youssef) on Mona's group | 403 `COURSE_NOT_ACCESSIBLE` |
| AU-03 | Secretary with an expired / suspended link | 403 from the teacher context |
| AU-04 | Group of another course in the URL | 404 `GROUP_NOT_FOUND` |
| AU-05 | No token | 401 |
| LO-01 | `GROUP_CHANGED` with `Accept-Language: ar` / `en` | Arabic / English message, same `error_code` |
| AD-01 | Admin panel → group page → «إجراءات المجموعة» | the six actions; each works and is audited with the admin |
| AD-02 | Support admin | the menu items are hidden |

## 8. Mobile checks

| ID | Check |
|---|---|
| MB-01 | One «الإجراءات» button (L25.1); the menu badge = inactive count. |
| MB-02 | Every confirm text uses the preview count; a 409 shows the new count and asks again. |
| MB-03 | Capacity problem shown before confirming (preview `available_seats`). |
| MB-04 | Exports download and open in the share sheet. |
| MB-05 | Activate: past dates disabled; done screen (L25.14) shows updated / unchanged. |

## 9. Bug report

Account, group, the request and full response (`error_code`, `errors`, `details`), headers sent (`Accept-Language`, `X-Teacher-Id`), the test ID and the time.
