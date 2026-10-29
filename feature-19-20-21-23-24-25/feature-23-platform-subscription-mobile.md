# Feature 23 — Platform subscription · Mobile integration guide

> For: mobile developers · API `v1` · technical reference: [feature-23-platform-subscription.md](../features/feature-23-platform-subscription.md) · QA guide: [feature-23-platform-subscription-qa.md](feature-23-platform-subscription-qa.md) · prerequisites: [Feature 02](feature-02-teacher-secretary-mobile.md) (teacher context)

The **teacher's subscription to Darsly** (not a student's course enrollment). The drawer shows «الخدمة هتنتهي بعد N يوم» with «تجديد». The teacher **requests** a renewal; an admin activates it after payment. Once the subscription has expired (after a 3-day grace), the teacher's and their secretaries' **write** actions answer `402`.

---

## 1. Common rules

* Base URL `{{host}}/api/v1`; send **`X-Teacher-Id`** (the teacher, or the teacher a secretary works for).
* Envelope `{code, message, errors, data}`; **branch on `error_code`**.

---

## 2. Status

| `status` | When | App |
|---|---|---|
| `active` | more than 30 days left | no bar (or a quiet one) |
| `expiring` | 30 days or less | bar «الخدمة هتنتهي بعد {days_left} يوم» + «تجديد» |
| `grace` | ended less than 3 days ago | red bar; everything still works |
| `expired` | after the grace | red bar; **writes return 402**, reads work |

`days_left` is already rounded by the server; never compute it on the phone.

---

## 3. Where the bar comes from (no extra call)

* `GET /me` → `user.roles[]` → the **teacher** role carries `platform_subscription {status, days_left, ends_at}` (`null` = no subscription yet).
* Secretary: `GET /secretary/teachers` → each `teacher.platform_subscription` (L1.3 «السكرتير · الاشتراك قرب يخلص»).

```json
{ "id": "019f…", "role": "teacher", "code": "77120458", "is_active": true,
  "platform_subscription": { "status": "expiring", "days_left": 22, "ends_at": "2026-11-16T12:00:00+00:00" } }
```

---

## 4. API

Base `/teacher/platform-subscription`:

| Action | Request | Success |
|---|---|---|
| Subscription | `GET /` | 200 |
| Plans | `GET /plans` | 200 |
| History (periods + requests) | `GET /periods` | 200 |
| Request a renewal | `POST /renewal-requests` | **201** |
| Cancel my pending request | `DELETE /renewal-requests/{requestId}` | 200 |

### 4.1 Subscription

```json
{ "plan": { "key": "monthly", "name": "شهري" }, "starts_at": "…", "ends_at": "2026-11-16T12:00:00+00:00",
  "days_left": 22, "status": "expiring", "blocks_writes": false, "can_renew": false,
  "pending_request": { "id": "019r…", "plan": { "key": "monthly", "name": "شهري" }, "note": "…", "status": "pending", "created_at": "…" } }
```

`can_renew: false` when a request is already pending, or for a secretary. `404 PLATFORM_SUBSCRIPTION_NOT_FOUND` = no subscription yet (accounts activated before this feature).

### 4.2 Plans

```json
{ "items": [ { "key": "monthly", "name": "شهري", "days": 30, "price": 500 }, { "key": "term", "name": "ترم", "days": 120, "price": 1800 }, { "key": "yearly", "name": "سنوي", "days": 365, "price": 5000 } ] }
```

Prices are for display (EGP); payment happens outside the app for now.

### 4.3 Request a renewal

```json
POST /teacher/platform-subscription/renewal-requests
{ "plan_key": "term", "note": "حوّلت المبلغ فودافون كاش" }
```

One pending request at a time (`409 RENEWAL_ALREADY_PENDING`). The admin approves (a new period starts after the current end, or now if already ended) or rejects with a reason (see `GET /periods` → `requests[].rejection_reason`).

---

## 5. Expired behaviour

* Any POST / PUT / PATCH / DELETE on teacher screens → **`402 PLATFORM_SUBSCRIPTION_EXPIRED`** (`details.ends_at`). Show a blocking sheet with «تجديد».
* Reads, these subscription endpoints, settings and logout keep working.
* Students are **not** affected.

---

## 6. Errors to handle

| `error_code` | HTTP | Show |
|---|---|---|
| `PLATFORM_SUBSCRIPTION_EXPIRED` | 402 | renewal sheet (handle globally for teacher writes) |
| `PLATFORM_SUBSCRIPTION_NOT_FOUND` | 404 | "no subscription yet — contact support" |
| `RENEWAL_ALREADY_PENDING` | 409 | "a request is waiting for approval" |
| `RENEWAL_REQUEST_NOT_FOUND` | 404 | already decided / cancelled; refresh |
| `INVALID_PLAN` | 422 | refresh plans |
| `RENEWAL_NOT_ALLOWED_FOR_SECRETARY` | 403 | only the teacher can renew |

---

## 7. Checklist

- [ ] Drawer bar built from `/me` (teacher) or `/secretary/teachers` (secretary), no extra call.
- [ ] `402` handled globally on teacher writes.
- [ ] Renew screen: plans, note, pending state with cancel.
- [ ] Secretary sees the status but no renew button.
- [ ] Push `data.type: platform_subscription` opens this screen ([Feature 20](feature-20-notifications-mobile.md)).
