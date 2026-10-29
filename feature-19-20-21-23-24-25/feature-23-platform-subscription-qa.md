# Feature 23 — Platform subscription · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-23-platform-subscription.md](../features/feature-23-platform-subscription.md) · mobile guide: [feature-23-platform-subscription-mobile.md](feature-23-platform-subscription-mobile.md)

---

## 1. Scope

* Status at each boundary (active, expiring, grace, expired), `days_left`.
* Plans, history, renewal requests (one pending, cancel), admin approve / reject, periods.
* 402 on management writes after the grace; reads and subscription endpoints still work; students unaffected.
* Drawer bar on `/me` and `/secretary/teachers`; reminders (7 / 3 / 1 days, pushed).
* First trial period on admin activation. Enrollment tables never touched.
* Admin panel: *Teaching → Platform subscriptions*.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

| Teacher | Subscription |
|---|---|
| Mona (`mona.mahmoud@darsly.test`) | yearly, **active** |
| «د. تسنيم سامي» (`tasneem.samy@darsly.test`) | **22 days left** + a **pending** monthly renewal |
| Hany Fathy (`hany.fathy@darsly.test`) | ended 10 days ago → **expired** |
| Youssef | no subscription (not blocked) |

Password `Darsly#User2026`. Postman: `postman/Darsly-Feature-23-Platform-Subscription.postman_collection.json`.

---

## 3. Test cases: status

| ID | Case | Expected |
|---|---|---|
| PS-01 | Mona | `active`, `can_renew: true` |
| PS-02 | Tasneem | `expiring`, `days_left: 22`, `pending_request` |
| PS-03 | Hany | `expired`, `blocks_writes: true` |
| PS-04 | Youssef | 404 `PLATFORM_SUBSCRIPTION_NOT_FOUND` |
| PS-05 | Boundaries (admin adjusts the end date): +31 d / +30 d / now / −2 d / −3 d | active / expiring / grace / grace / expired |
| PS-06 | `/me` as Tasneem | teacher role `platform_subscription {expiring, 22, ends_at}` |
| PS-07 | `/secretary/teachers` as Ahmed | each teacher with its bar |

## 4. Test cases: renewal

| ID | Case | Expected |
|---|---|---|
| PR-01 | Mona requests `term` | 201 pending; `can_renew: false` |
| PR-02 | Again | 409 `RENEWAL_ALREADY_PENDING` |
| PR-03 | `plan_key: trial` / unknown | 422 `INVALID_PLAN` |
| PR-04 | Cancel own request | 200; request `cancelled`; can request again |
| PR-05 | Another teacher cancels it | 404 `RENEWAL_REQUEST_NOT_FOUND` |
| PR-06 | Secretary requests | 403 `RENEWAL_NOT_ALLOWED_FOR_SECRETARY` |
| PR-07 | Admin approves Tasneem's (22 days left, monthly) | new period starts at her current end; `ends_at` + 30 days |
| PR-08 | Admin approves for Hany (expired) | the period starts **now** |
| PR-09 | Admin rejects with a reason | `periods.requests[].status: rejected`, `rejection_reason` |
| PR-10 | History | periods newest first with `source` (`trial`, `activation`, `renewal`, `admin_adjustment`) |
| PR-11 | Enrollment transactions | none created by any of the above |

## 5. Test cases: expired behaviour

| ID | Case | Expected |
|---|---|---|
| PE-01 | Hany creates a course / group / lecture | 402 `PLATFORM_SUBSCRIPTION_EXPIRED` (`details.ends_at`) |
| PE-02 | Hany lists his courses | 200 |
| PE-03 | Hany requests a renewal | 201 (allowed) |
| PE-04 | A secretary of Hany writes | 402 |
| PE-05 | A student of Hany opens lectures | works (students unaffected) |
| PE-06 | Grace (ended 1 day ago) | writes still work |

## 6. Test cases: activation & reminders

| ID | Case | Expected |
|---|---|---|
| PA-01 | Admin activates a new teacher | a 30-day `trial` period; status `expiring` |
| PA-02 | `php artisan darsly:platform:remind-expiring` with subscriptions at 7 / 3 / 1 / 5 days | 3 reminders (logged + pushed to the teacher) |

## 7. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Platform subscriptions list | teacher, plan, state, days left, end date, pending renewal; filters by state / pending |
| AD-02 | Add period (header: any teacher; row) | period added; audit `platform_subscription.period_added` |
| AD-03 | Adjust end date | `admin_adjustment` period; may shorten |
| AD-04 | Approve / reject | as PR-07 / PR-09 |
| AD-05 | Support admin | sees the list, no actions |

## 8. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Bar «الخدمة هتنتهي بعد 22 يوم» for Tasneem. |
| MB-02 | 402 opens the renewal sheet anywhere in teacher mode. |
| MB-03 | Secretary sees the bar without the renew button. |

## 9. Bug report

Teacher, `X-Teacher-Id`, subscription end date, request and full response (`error_code`, `details`), the test ID and the time.
