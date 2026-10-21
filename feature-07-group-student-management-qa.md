# Feature 07 — Group Students · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-07-group-student-management.md](../features/feature-07-group-student-management.md) · mobile guide: [feature-07-group-student-management-mobile.md](feature-07-group-student-management-mobile.md) · related: [Feature 05 QA](feature-05-students-enrollment-qa.md), [Feature 06 QA](feature-06-enrollment-transactions-qa.md)

---

## 1. Scope

* The roster of one group: statuses, search, status and expiry filters, pagination; `students_count` on the groups list.
* Add one or many students (new, restored, existing), shared defaults and per-student overrides, per-student images and payment method.
* Move, bulk move, add to another group, **bulk add to another group**.
* **Bulk status**.
* Remove, bulk remove, the last-group rule.
* All-or-nothing behaviour and per-item error keys.
* Access: the course's teacher or a validly linked secretary.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

Accounts as in the [Feature 05 QA guide](feature-05-students-enrollment-qa.md#2-setup--data). Seeded Mathematics roster (Mona, automatic transactions **ON**):

| Group | Members | Notes |
|---|---|---|
| المجموعة A (300.00, capacity 50) | Omar (active), Nour (active) | |
| المجموعة B (no price, capacity 30) | Sara (**suspended**) | |
| المجموعة المسائية (free, unlimited) | Laila (active), Nour (active) | |
| Physics: مجموعة الصيفي (متوقفة) | — | **inactive** group of another course |

Get ids from `GET /teacher/courses/{id}/groups`; enrollment ids from the roster. Postman: `postman/Darsly-Feature-07-Group-Student-Management.postman_collection.json` — run folders in order; folder 09 restores the seeded roster.

---

## 3. Test cases: groups list & roster

| ID | Case | Expected |
|---|---|---|
| RL-01 | `GET /teacher/courses/{math}/groups` | each group has `students_count`: A = 2, B = 1, evening = 2 |
| RL-02 | Remove a member, list groups again | the count drops by 1 (removed memberships aren't counted) |
| RL-03 | `GET …/groups/{A}` (single) | no `students_count` |
| RL-04 | Roster of A | Omar and Nour; `groups` holds only A; no amounts, email, images |
| RL-05 | `?status=suspended` on B | Sara |
| RL-06 | `?search=` part of a name / code / phone | matches |
| RL-07 | `?expiry=expiring&expiring_within_days=30` on A (before 2026-11-01) | Omar (ends 2026-11-01); Nour (no end date) is not listed |
| RL-08 | `?expiry=expired` | only students whose end has passed |
| RL-09 | `?expiring_within_days=3` alone; `?expiry=soon` | 422 |
| RL-10 | `?per_page=1&page=2` | one item, page 2 |
| RL-11 | A Physics group id in the Math URL | 404 `GROUP_NOT_FOUND` |

## 4. Test cases: add students

`POST …/groups/{group}/students`.

| ID | Case | Expected |
|---|---|---|
| AS-01 | One new student by code into A, `amount: 300, payment_method: "cash"` | **201**; `operation: created`; `enrollment` 300.00 with `cash` |
| AS-02 | The same student again | 201; `operation: skipped`, no transaction; other students in the same request are still added |
| AS-03 | Bulk into evening: a new student + Omar (existing), `defaults {amount: 150, payment_method: "cash"}`, Omar sends `amount: null` | new: `created`, 150.00 / cash; Omar: `attached`, **no** renewal (his own amount is null; defaults don't apply to existing enrollments) |
| AS-04 | Per-student method override | defaults `cash`, one student `instapay` | that student's transaction is `instapay` |
| AS-05 | Unknown method | `payment_method: "bitcoin"` (defaults or a student) | 422 |
| AS-06 | Existing student with own amount + images (multipart) | `attached` + a `renewal` with the images |
| AS-07 | `defaults.images` | 422 |
| AS-08 | Sara (suspended) added to evening | re-activated (`status_changed` reason `group_added`) |
| AS-09 | Restore | a student removed from the course earlier | `restored`; old groups not back |
| AS-10 | All or nothing | 3 students, the 3rd already in the group | 422 (`students.2`); the first two were **not** added |
| AS-11 | Duplicate | the same student by id and by code | 422 `DUPLICATE_STUDENT_SELECTION` |
| AS-12 | Unknown code | | 404 `STUDENT_NOT_FOUND` (`students.N`) |
| AS-13 | Shape | empty `students`; both id and code; 51 students; negative amount; unknown status | 422 |
| AS-14 | Window from defaults + override | defaults start / a student's end before it | 422 `INVALID_SUBSCRIPTION_DATES`; nothing written |
| AS-15 | Inactive / full group | add into an inactive group; a group at capacity | 422 `GROUP_INACTIVE`; 409 `GROUP_CAPACITY_REACHED` |

## 5. Test cases: move & add to another group

| ID | Case | Expected |
|---|---|---|
| MV-01 | Move Omar A → B | 200 `moved`; groups = B; status unchanged; `group_removed` + `group_added` |
| MV-02 | Add Omar (in B) to A | `added`; groups = A + B; `group_added` only |
| MV-03 | Target = source | 422 `TARGET_GROUP_SAME_AS_SOURCE` |
| MV-04 | Target of another course / unknown | 422 `INVALID_GROUP_ASSIGNMENT` |
| MV-05 | Target inactive / full | 422 `GROUP_INACTIVE` / 409 `GROUP_CAPACITY_REACHED` |
| MV-06 | Student not in the source | 422 `STUDENT_NOT_IN_GROUP` |
| MV-07 | Already in the target | 200; `operation: skipped`, no transaction; the student stays in the source **and** the target |
| MV-08 | Suspended / expelled moved | status preserved (never re-activated) |
| MV-09 | Active student moved from their only group | stays active (ends in the active target) |
| MV-10 | Bulk move, one invalid item | 422 naming `course_enrollment_ids.N`; nobody moved |
| MV-11 | Bulk move: duplicate ids; no target | 422 |

## 6. Test cases: bulk add to another group

`POST …/groups/{source}/students/bulk-add-to-group`.

| ID | Case | Body | Expected |
|---|---|---|---|
| BA-01 | Two members of A → B | `{course_enrollment_ids: [Omar, Nour], target_group_id: B}` | 200; both `added`; both keep A and get B; statuses unchanged; a `group_added` each |
| BA-02 | One already in the target | Nour → evening | 200; Nour `skipped` (untouched), the others `added` |
| BA-03 | One not in the source | Sara (B only) from A | 422 `STUDENT_NOT_IN_GROUP`; nobody added |
| BA-04 | No target | | 422 (`target_group_id`) |
| BA-05 | Target = source / other course / inactive / full | | the same errors as MV-03 – MV-05 |
| BA-06 | Suspended student | | stays suspended |
| BA-07 | Youssef (OFF) | his own course | memberships change; no `group_added` rows |

## 7. Test cases: bulk status

`POST …/groups/{group}/students/bulk-status`.

| ID | Case | Body | Expected |
|---|---|---|---|
| BS-01 | Suspend two | A: `[Omar, Nour]`, `suspended` | 200; both `status_changed`; `status_changed` rows with `reason: explicit`; groups untouched |
| BS-02 | Already equal | the same request again | both `unchanged`; `transactions: []`; no new rows |
| BS-03 | Mixed | one active + one suspended → `suspended` | `status_changed` + `unchanged` |
| BS-04 | Re-activate | `active` | both active (they have active groups) |
| BS-05 | Active without an active group | a student whose only group is inactive, inside that group | 422 `ACTIVE_ENROLLMENT_REQUIRES_ACTIVE_GROUP` naming that item; the others unchanged |
| BS-06 | Expel | `expelled` | expelled; `access_reason: expelled` on the enrollment |
| BS-07 | Non-member | an enrollment of another group | 422 `STUDENT_NOT_IN_GROUP` |
| BS-08 | Shape | no `status`; `status: "graduated"`; duplicate ids; 51 ids | 422 |
| BS-09 | `status` on other endpoints | `status` on bulk-remove / bulk-move | 422 (`status`) |
| BS-10 | Secretary | Ahmed in Mona's context | 200; rows `created_by.role: secretary` |
| BS-11 | Youssef (OFF) | his group | statuses change; no `status_changed` rows |

## 8. Test cases: remove

| ID | Case | Expected |
|---|---|---|
| RM-01 | Remove Nour from A | 200 `removed`; she keeps evening; stays active |
| RM-02 | Remove an active student from their only active group | `suspended` + `status_changed` (`last_active_group_removed`) |
| RM-03 | Remove a suspended / expelled student | status preserved, no `status_changed` |
| RM-04 | Remove a non-member | 422 `STUDENT_NOT_IN_GROUP` |
| RM-05 | Bulk remove, one invalid | 422 naming the item; nobody removed |
| RM-06 | History kept | the student's payments and history are unchanged (Feature 06) |
| RM-07 | Re-add | the removed membership row is restored (`group_added` operation `restored`) |

## 9. Authorization & context

| ID | Case | Expected |
|---|---|---|
| AU-01 | Ahmed (linked) lists and writes | allowed; rows under the secretary role |
| AU-02 | Youssef on Mona's course (every write incl. bulk-add-to-group and bulk-status) | 403 `COURSE_NOT_ACCESSIBLE` |
| AU-03 | Nour in Youssef's context (expired link) | 403 `SECRETARY_RELATION_EXPIRED` |
| AU-04 | Ahmed in Youssef's context (suspended link) | 403 `SECRETARY_RELATION_INACTIVE` |
| AU-05 | Laila (inactive secretary) | 403 |
| AU-06 | An enrollment of another teacher in the URL / body | 403 `ENROLLMENT_NOT_ACCESSIBLE` |
| AU-07 | Enrollment of another course of the same teacher | 422 `STUDENT_NOT_IN_GROUP` |
| AU-08 | Missing `X-Teacher-Id`; unauthenticated | 422; 401 |
| AU-09 | Arabic / English | same `error_code`, translated messages (e.g. "Students status updated successfully.") |

## 10. Admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | *Teaching → Groups → a group → Students* | the roster; add, move, add to another group, remove — same rules as the API |
| AD-02 | Bulk add to another group / bulk status | **API only** for now (not in the admin panel) |
| AD-03 | Payment method on add | not offered in the admin form yet (stored as `null`) |

## 11. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Group cards show `students_count`. |
| MB-02 | Multi-select offers move, add to group, status and remove. |
| MB-03 | A bulk error highlights the named row; the list is unchanged after the error. |
| MB-04 | Target pickers hide the current group and inactive groups. |
| MB-05 | After remove / bulk status, the row status comes from the response. |
| MB-06 | "Ending soon" / "Ended" filters work on the roster. |

## 12. Bug report

Account, `X-Teacher-Id`, course / group / enrollment ids, the request and full response (`error_code`, `errors` with the item key), the test ID and the time. No tokens or passwords.
