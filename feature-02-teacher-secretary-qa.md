# Feature 02 — Teacher ↔ Secretary · QA test guide

> For: testers (manual API, mobile app and admin panel) · API `v1` · technical reference: [feature-02-teacher-secretary.md](../features/feature-02-teacher-secretary.md) · mobile guide: [feature-02-teacher-secretary-mobile.md](feature-02-teacher-secretary-mobile.md)

---

## 1. What is being tested

1. A **teacher** links secretaries by their **secretary code**. They can then list, extend (`valid_until`), suspend, reactivate and remove those links.
2. A **secretary** lists the teachers they work for, picks one, and works in that teacher's context by sending `X-Teacher-Id`.
3. The **teacher context** check on every `/teacher/...` endpoint: who may act for which teacher.
4. The **admin panel** screen *Teaching → Secretary links*, where the admin can do the same link operations for any teacher.

---

## 2. Environment & test data

```bash
php artisan migrate:fresh --seed     # local only: resets and seeds all demo data
```

Password for every seeded account: **`Darsly#User2026`**. Admin panel: `/admin`.

| Account | Use it as | Notes |
|---|---|---|
| `mona.mahmoud@darsly.test` | **Teacher A** | teacher role active |
| `youssef.ibrahim@darsly.test` | **Teacher B** | teacher role active |
| `ahmed.hassan@darsly.test` | **Secretary A** | his own **teacher** role is inactive |
| `nour.salem@darsly.test` | **Secretary B** | |
| `laila.fathy@darsly.test` | inactive secretary | secretary role **inactive**, so she can't be attached |
| `omar.khaled@darsly.test` | detached secretary of Mona | Google account (no password) |
| `sara.ali@darsly.test` | incomplete profile | for the `PROFILE_INCOMPLETE` case |
| `admin@darsly.test` | super admin (panel) | full access |

Seeded links:

| Link | State |
|---|---|
| Mona ↔ Ahmed | active, never expires |
| Mona ↔ Nour | active, expires in 90 days |
| Youssef ↔ Ahmed | **suspended** |
| Youssef ↔ Nour | active but **expired** (7 days ago) |
| Mona ↔ Omar | **removed** (soft-deleted) |

**Getting ids and codes:** log in (`POST /api/v1/auth/login`) or call `GET /api/v1/me`. `data.user.roles[]` has, for each role, `id` (UUID, used as `X-Teacher-Id`), `code` (digits, used to attach) and `is_active`.

A ready-made Postman collection exists: `postman/Darsly-Feature-02-Teacher-Secretary.postman_collection.json` (run it in order; wait ~60 s between full runs because of rate limits).

**Every response** has the shape `{code, message, errors, data}`; errors add `error_code`. Check **HTTP status + `error_code`**, and check that `message` is in the requested language (`Accept-Language: ar|en`).

---

## 3. Test cases: teacher manages secretaries

Headers: Mona's token + `X-Teacher-Id` = Mona's teacher role id, unless the case says otherwise.

| ID | Case | Steps | Expected |
|---|---|---|---|
| TS-01 | List my secretaries | `GET /teacher/secretaries` | 200; Ahmed (active) and Nour (with `valid_until`); Omar **not** listed (removed); only **secretary** codes shown |
| TS-02 | Pagination | `?per_page=1&page=2` | 200; 1 item; `pagination.current_page = 2` |
| TS-03 | Page size cap | `?per_page=1000` | 422 `VALIDATION_ERROR` |
| TS-04 | Attach new | `POST {code: <a secretary code not linked to Mona>, valid_until: null}` | **201**; link `is_active: true`, `is_expired: false` |
| TS-05 | Attach with expiry | same with a future `valid_until` | 201; `valid_until` returned |
| TS-06 | Restore removed | `POST {code: <Omar's secretary code>}` | **200**; same link id as before, active again |
| TS-07 | Already linked | `POST {code: <Ahmed's secretary code>}` | 409 `SECRETARY_ALREADY_ATTACHED` |
| TS-08 | Unknown code | `POST {code: "99999999"}` | 404 `SECRETARY_NOT_FOUND` |
| TS-09 | Teacher / student code | `POST` with a **teacher** code, then a **student** code | both 404 `SECRETARY_NOT_FOUND`, identical message |
| TS-10 | Inactive secretary | `POST {code: <Laila's secretary code>}` | 404 `SECRETARY_NOT_FOUND` (same answer as unknown) |
| TS-11 | Self attach | `POST {code: <Mona's own secretary code>}` | 422 `CANNOT_ATTACH_SELF_AS_SECRETARY` |
| TS-12 | Validation | `code: "abc"`; `valid_until` in the past; no `code` | 422 `VALIDATION_ERROR` with field keys |
| TS-13 | Ignored fields | `POST {code, is_active: false, teacher_user_role_id: <Youssef>}` | link created for **Mona**, **active** |
| TS-14 | Extend | `PATCH /teacher/secretaries/{id} {valid_until: <future>}` | 200; new date |
| TS-15 | Remove expiry | `PATCH … {valid_until: null}` | 200; `valid_until: null` |
| TS-16 | Expiry required | `PATCH … {}` | 422 `VALIDATION_ERROR` |
| TS-17 | Past expiry | `PATCH … {valid_until: <past>}` | 422 `VALIDATION_ERROR` |
| TS-18 | Suspend | `PATCH …/deactivate` | 200 `is_active: false`; a second call is also 200 (no change) |
| TS-19 | Reactivate | `PATCH …/activate` | 200 `is_active: true` |
| TS-20 | Reactivate expired | as **Youssef** on the Youssef↔Nour link: `PATCH …/activate` | 422 `RELATION_EXPIRED`; then `PATCH {valid_until: future}` and activate again → 200 |
| TS-21 | Remove | `DELETE /teacher/secretaries/{id}` | 200; link gone from both lists |
| TS-22 | Remove twice | repeat TS-21 | 404 `SECRETARY_RELATION_NOT_FOUND` |
| TS-23 | Bad id | `PATCH /teacher/secretaries/not-a-uuid/activate`; a random UUID | 404 |
| TS-24 | Attach rate limit | 11 attach attempts within a minute (any codes) | attempts 1–10 answer normally; the 11th → 429 `TOO_MANY_REQUESTS` with `Retry-After` |
| TS-25 | Manage rate limit | 31 PATCH/DELETE calls within a minute | 31st → 429 |
| TS-26 | Localization | TS-08 with `Accept-Language: ar`, then `en` | same `error_code`; Arabic, then English `message` |

## 4. Test cases: security (IDOR & role checks)

| ID | Case | Steps | Expected |
|---|---|---|---|
| SC-01 | Other teacher's link | as **Mona** (own context): `PATCH /teacher/secretaries/{a Youssef link id}/deactivate` | 403 `TEACHER_CONTEXT_FORBIDDEN`; link unchanged |
| SC-02 | Same for update / activate / delete | repeat SC-01 for each operation | 403 each time; nothing changes |
| SC-03 | Spoof the context | as **Mona** with `X-Teacher-Id` = **Youssef**'s teacher id | 403 `SECRETARY_NOT_LINKED_TO_TEACHER` |
| SC-04 | Secretary can't manage links | as **Ahmed** with `X-Teacher-Id` = Mona: list / attach / deactivate / delete | 403 `TEACHER_CONTEXT_FORBIDDEN` for all |
| SC-05 | Role spoofing | send `role`, `acting_as`, `actor_role_id` in body / query / extra headers | ignored; result decided by the database only |
| SC-06 | No token | any endpoint without `Authorization` | 401 `UNAUTHENTICATED` |
| SC-07 | Blocked account | admin blocks Ahmed's login (Users → Block login); Ahmed retries with his old token, then tries to log in again | old token → 401 `UNAUTHENTICATED` (blocking revokes all sessions); new login → 403 `LOGIN_NOT_ALLOWED`. Allow login again afterwards |

## 5. Test cases: secretary picks a teacher

| ID | Case | Steps | Expected |
|---|---|---|---|
| SE-01 | List my teachers | as **Ahmed**: `GET /secretary/teachers` (**no** `X-Teacher-Id`) | 200; Mona `is_usable: true`; Youssef `is_active: false`, `is_usable: false` |
| SE-02 | Expired shown | as **Nour** | Mona usable; Youssef `is_expired: true`, `is_usable: false` |
| SE-03 | Removed not shown | as **Omar** (Google sign-in only: test through the app, or skip; covered by automated tests) | Mona **not** listed |
| SE-04 | Use the id | take `teacher.role_id` of Mona from SE-01; call `GET /teacher/course-enrollments` with it as `X-Teacher-Id` | 200 (secretary acts for Mona) |
| SE-05 | Inactive secretary role | as **Laila** | 403 `SECRETARY_ROLE_NOT_ACTIVE` |
| SE-06 | Only the teacher's code | inspect SE-01 items | only `teacher.code`; no student or secretary codes of the teacher |

## 6. Test cases: teacher context on any teacher endpoint

Use any teacher-context endpoint, e.g. `GET /api/v1/teacher/courses`.

| ID | Case | Actor + `X-Teacher-Id` | Expected |
|---|---|---|---|
| CX-01 | Header missing | Mona, no header | 422 `TEACHER_CONTEXT_HEADER_REQUIRED` |
| CX-02 | Not a UUID | Mona, `X-Teacher-Id: 32883` (a code) | 422 `INVALID_TEACHER_CONTEXT` |
| CX-03 | Unknown teacher | Mona, random UUID | 404 `TEACHER_NOT_FOUND` |
| CX-04 | Non-teacher role id | Mona, her **student** role id | 404 `TEACHER_NOT_FOUND` |
| CX-05 | Own context | Mona, Mona | 200 |
| CX-06 | Inactive own teacher role | Ahmed, Ahmed's **teacher** role id | 403 `TEACHER_ROLE_NOT_ACTIVE` |
| CX-07 | Valid secretary | Ahmed, Mona | 200 |
| CX-08 | Suspended link | Ahmed, Youssef | 403 `SECRETARY_RELATION_INACTIVE` |
| CX-09 | Expired link | Nour, Youssef | 403 `SECRETARY_RELATION_EXPIRED` |
| CX-10 | Removed link | Mona removes Nour (TS-21), then Nour → Mona | 403 `SECRETARY_NOT_LINKED_TO_TEACHER`; re-attach Nour's code afterwards (restores the link) |
| CX-11 | No link at all | Youssef, Mona | 403 `SECRETARY_NOT_LINKED_TO_TEACHER` |
| CX-12 | Immediate effect | Mona suspends Ahmed (TS-18), then Ahmed repeats CX-07 | the **next** request → 403 `SECRETARY_RELATION_INACTIVE` (no logout needed); reactivate → 200 again |
| CX-13 | Expiry by time | Mona sets Ahmed's `valid_until` to ~2 minutes ahead; wait; repeat CX-07 | 200 before, 403 `SECRETARY_RELATION_EXPIRED` after (nothing else changes) |
| CX-14 | Admin deactivates teacher | admin deactivates Mona's teacher role; Ahmed repeats CX-07 | 403 `TEACHER_ROLE_NOT_ACTIVE`; reactivate Mona afterwards |
| CX-15 | Incomplete profile | `sara.ali@darsly.test` (profile not completed), any `X-Teacher-Id` | 403 `PROFILE_INCOMPLETE` |

## 7. Test cases: admin panel (Teaching → Secretary links)

Log in to `/admin` as `admin@darsly.test`.

| ID | Case | Steps | Expected |
|---|---|---|---|
| AD-01 | See all links | open *Secretary links* | every teacher's links with **State**: Active / Suspended / Expired; the filter *Detached links* also shows removed ones |
| AD-02 | Attach for a teacher | *Attach a secretary* → pick Teacher B, enter Secretary A's code | success notification; the link appears; in the API, Youssef's `GET /teacher/secretaries` shows it |
| AD-03 | Same rules as the API | attach an unknown code, a self code, an already-linked code | a red notification with the same reason as TS-08 / TS-11 / TS-07; nothing is created |
| AD-04 | Suspend / Activate / Change expiry / Detach | row menu actions | same effects as TS-14…TS-21; activating an expired link shows the `RELATION_EXPIRED` notification |
| AD-05 | Audit trail | after AD-02, look in `audit_logs` (or ask a developer) | the actor is the **admin**; the metadata contains `on_behalf_of_admin_id` |
| AD-06 | Read-only admin | log in as a **Support** admin | the list is visible; *Attach* and the row actions are **hidden** |
| AD-07 | Effect on the secretary | suspend Mona ↔ Ahmed from the panel | Ahmed's next teacher request → 403 `SECRETARY_RELATION_INACTIVE` |

## 8. Mobile app checks

| ID | Check |
|---|---|
| MB-01 | Teacher mode sends the **own teacher role id**; a not-yet-activated teacher sees a waiting state, not an error loop. |
| MB-02 | Secretary mode starts with the teacher picker; unusable teachers are greyed out with the right reason (suspended / expired / teacher inactive). |
| MB-03 | After the teacher suspends or removes the secretary (CX-12), the app shows a message and returns to the picker on the next action. |
| MB-04 | "Add secretary" treats **201 and 200** as success (TS-04, TS-06) and shows the API reason for TS-07…TS-11. |
| MB-05 | A 429 shows a wait message using `Retry-After`; the button is not spammable. |
| MB-06 | Arabic and English UI show the translated `message`; behaviour does not depend on the language. |
| MB-07 | The secretary-management screen is not reachable in secretary mode (SC-04). |
| MB-08 | Dates entered for `valid_until` keep the device timezone (the value shown after saving equals the value entered). |

## 9. Regression after changes

Re-run the Postman collection (all green), then spot-check TS-04, TS-06, SC-01, SE-01, CX-07…CX-10 and AD-02.

## 10. Reporting a bug

Include:

* the account and role used;
* the exact request: method, URL, headers (`X-Teacher-Id`, `Accept-Language`) and body;
* the full response: HTTP status, `error_code`, `message`, `errors`;
* the expected result (test case ID);
* the time, which matters for expiry cases.

Never paste access tokens or passwords into tickets.
