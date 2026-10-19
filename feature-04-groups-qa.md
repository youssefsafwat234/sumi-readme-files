# Feature 04 — Groups · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-04-groups.md](../features/feature-04-groups.md) · mobile guide: [feature-04-groups-mobile.md](feature-04-groups-mobile.md)

---

## 1. Scope

* Groups inside a course: create, list (search, active filter, pagination), details, edit (fields, activate / deactivate, image), delete.
* Defaults stored on a group: end date, price (null vs free), capacity (null = unlimited).
* Access: the course's teacher, or a linked secretary in that teacher's context; nobody else; no cross-course access.
* Admin panel: *Teaching → Groups* and the *Groups* tab of a course.
* Links to later features: inactive or full groups can't receive students (Features 6–7).

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

Password `Darsly#User2026`; admin `admin@darsly.test` at `/admin`. Accounts as in the Feature 3 guide: Mona (Teacher A), Youssef (Teacher B), Ahmed and Nour (Mona's secretaries).

Seeded groups (end dates are relative to the seed day):

| Course (teacher) | Group | End date | Price | Capacity | Active |
|---|---|---|---|---|---|
| مراجعة الرياضيات (Mona) | المجموعة A | +1 month | 300.00 | 50 | ✓ |
| مراجعة الرياضيات (Mona) | المجموعة B | +30 days | *no price* | 30 | ✓ |
| مراجعة الرياضيات (Mona) | المجموعة المسائية | *none* | 0.00 (free) | unlimited | ✓ |
| شرح الفيزياء (Mona) | مجموعة الجمعة | +3 months | 850.00 | 25 | ✓ |
| شرح الفيزياء (Mona) | مجموعة الصيفي (متوقفة) | +14 days | 120.50 | unlimited | ✗ |
| Youssef's courses | المجموعة الأولى, مجموعة المنح, المجموعة A … | | | | one inactive |

Get course ids from `GET /teacher/courses`. Postman: `postman/Darsly-Feature-04-Groups.postman_collection.json`.

---

## 3. Test cases: create

As Mona (own context) on her Math course, unless stated.

| ID | Case | Body | Expected |
|---|---|---|---|
| GC-01 | Minimal | `{name}` | **201**; `is_active: true`; end date / price / capacity `null` |
| GC-02 | Full | name, description, end date (future), price `300`, capacity `50` | 201; `subscription_price: "300.00"` (string) |
| GC-03 | Free vs no price | `subscription_price: 0`; another group without a price | `"0.00"` vs `null`, kept distinct in list and details |
| GC-04 | Price rules | `-1`, `10.555`, `"abc"` | 422 `VALIDATION_ERROR` (`subscription_price`) |
| GC-05 | End date today | today's date | 201 |
| GC-06 | End date rules | yesterday; `31/01/2027`; `2027-02-30`; a value with a time | 422 (`default_subscription_ends_at`) |
| GC-07 | Capacity rules | `0`, `-5`, `1.5`, `100001` | 422 (`max_students`) |
| GC-08 | Name rules | missing, `"A"`, 256 characters | 422 (`name`) |
| GC-09 | `is_active` on create | `{name, is_active: false}` | 201, but `is_active: **true**` (ignored) |
| GC-10 | Image | jpg ≤ 2 MB / 3 MB / pdf | 201 with URL / 422 / 422 |
| GC-11 | Same name twice | create "Group A" again | 201 (names don't have to be unique) |
| GC-12 | Secretary | Ahmed with `X-Teacher-Id` = Mona | 201; the group belongs to Mona's course |
| GC-13 | Another teacher's course | Mona creates in a Youssef course | 403 `COURSE_NOT_ACCESSIBLE` |
| GC-14 | Unknown / deleted course | random UUID / a deleted course | 404 `COURSE_NOT_FOUND` |
| GC-15 | Suspended / expired secretary | Ahmed → Youssef; Nour → Youssef | 403 `SECRETARY_RELATION_INACTIVE` / `SECRETARY_RELATION_EXPIRED` |

## 4. Test cases: list & details

| ID | Case | Expected |
|---|---|---|
| GL-01 | List Math groups | 3 groups, all fields present |
| GL-02 | `?is_active=0` on Physics | only the summer (inactive) group |
| GL-03 | `?is_active=1` | active groups only |
| GL-04 | `?search=المسائية` | the evening group only; `%` / `_` are treated literally |
| GL-05 | Pagination `?per_page=1&page=2`; `?per_page=101` | 1 item; 422 |
| GL-06 | Details | `GET …/groups/{id}` → 200 |
| GL-07 | Group of **another course** of the same teacher | Math course URL + a Physics group id → 404 `GROUP_NOT_FOUND` |
| GL-08 | Group of another teacher | → 403 `COURSE_NOT_ACCESSIBLE` (on their course) or 404 `GROUP_NOT_FOUND` (inside your own course URL) |
| GL-09 | Deleted group | 404 `GROUP_NOT_FOUND` |
| GL-10 | Non-UUID ids | `/teacher/courses/abc/groups`; `…/groups/abc` | 404 (route) |

## 5. Test cases: edit

| ID | Case | Expected |
|---|---|---|
| GU-01 | Change price only | `PATCH {subscription_price: 350}` → `"350.00"`; other fields unchanged |
| GU-02 | Clear fields | `PATCH {default_subscription_ends_at: null, subscription_price: null, max_students: null, description: null}` → all `null` |
| GU-03 | Omit vs null | `PATCH {name}` leaves price / date / capacity as they were |
| GU-04 | Past date kept | a group whose stored date is now in the past: `PATCH {name}` → 200 (date kept); `PATCH {default_subscription_ends_at: <yesterday>}` → 422 |
| GU-05 | Deactivate | `PATCH {is_active: false}` → 200; still listed with `?is_active=0` |
| GU-06 | Activate | `PATCH {is_active: true}` → 200 |
| GU-07 | Image | `POST` multipart + `_method=PATCH` → new URL |
| GU-08 | Capacity below members | set `max_students` lower than the current members (Feature 6 data) | 200 (allowed); existing members stay; new additions → 409 `GROUP_CAPACITY_REACHED` |
| GU-09 | Defaults don't touch students | change the end date / price of a group with students | existing enrollments keep their own dates and amounts |
| GU-10 | Another teacher / course | as in GL-07 / GL-08 | 404 / 403; nothing changes |

## 6. Test cases: delete

| ID | Case | Expected |
|---|---|---|
| GD-01 | Delete | 200 `data: null`; gone from lists; `GET` → 404 `GROUP_NOT_FOUND` |
| GD-02 | Delete twice | 404 `GROUP_NOT_FOUND` |
| GD-03 | Delete a group with students | 200; the students' enrollments stay; the group no longer appears in their groups. **Current behaviour:** the enrollment status is **not** changed automatically (an active student whose only group was deleted keeps `active` but has no access, per Feature 6 access rules). Report this if the business expects auto-suspension. |
| GD-04 | Delete in another teacher's course | 403 `COURSE_NOT_ACCESSIBLE` |

## 7. Test cases: effect on enrollments (Features 6–7)

| ID | Case | Expected |
|---|---|---|
| GE-01 | Enroll into an inactive group | 422 `GROUP_INACTIVE` |
| GE-02 | Group full | a group with `max_students: 1` and one member; add another | 409 `GROUP_CAPACITY_REACHED` |
| GE-03 | Unlimited | `max_students: null`; add many students | all accepted |
| GE-04 | Suggested amount | `GET /teacher/course-enrollments/suggested-amount?course_id=&group_ids[]=` | the sum of the selected groups' prices; incomplete when one group has no price |

## 8. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Teaching → Groups | every teacher's groups with course, instructor, price, capacity, member count; filters by course / instructor / status |
| AD-02 | Create from a course page | Courses → a course → *Groups* tab → *New group* | created in that course, for that teacher (audit actor = the admin) |
| AD-03 | Same validation as the API | a negative price, a past end date | form error or red notification; nothing saved |
| AD-04 | Edit / Activate / Deactivate / Delete | the row menu | same effects as GU / GD |
| AD-05 | Students tab | open a group | its students (Feature 7): add, move, add to another group, remove |
| AD-06 | Support admin | read-only: no create / edit / delete buttons |

## 9. Mobile checks

| ID | Check |
|---|---|
| MB-01 | "Free" (`0.00`) vs "No price" (`null`) are displayed differently. |
| MB-02 | "Unlimited" is shown for `max_students: null`. |
| MB-03 | The end-date picker blocks past days and can clear the date. |
| MB-04 | Editing one field doesn't clear the others. |
| MB-05 | Inactive groups have a badge and don't appear in "add students to group" pickers. |
| MB-06 | Secretary mode works on the picked teacher's courses only. |

## 10. Bug report

Account, `X-Teacher-Id`, course id, group id, the request and the full response (`error_code`, `errors`), the test ID, and the time (dates matter). No tokens or passwords.
