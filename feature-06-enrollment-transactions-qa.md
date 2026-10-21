# Feature 06 — Payments, Renewals & History · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-06-enrollment-transactions.md](../features/feature-06-enrollment-transactions.md) · mobile guide: [feature-06-enrollment-transactions-mobile.md](feature-06-enrollment-transactions-mobile.md) · enrollment cases: [Feature 05 QA](feature-05-students-enrollment-qa.md)

---

## 1. Scope

* Manual transactions: `renewal` (default) and `enrollment`, with amount, payment method, message, transfer images and date changes.
* Payment summary on the enrollment: `last_payment`, `total_paid`, `currency`.
* History: listing, filters, one transaction, the author, the date snapshot.
* Editing a financial transaction in place; transfer images (add, remove, download); no deletion.
* The per-teacher *automatic transactions* setting (admin) and `GET /teacher/settings`.
* Access: the course's teacher or a validly linked secretary; students never see any of it.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

Accounts and seeded enrollments as in the [Feature 05 QA guide](feature-05-students-enrollment-qa.md#2-setup--data). For payments:

| Enrollment (Mona's Mathematics) | Seeded history |
|---|---|
| Omar | `enrollment` 300.00 · `group_added` · `renewal` 300.00 |
| Sara | `enrollment` (no amount) · `group_added` |
| Laila | `enrollment` 0.00 (recorded by the secretary Ahmed) · `group_added` |
| Nour | `enrollment` 300.00 (by Ahmed) · `group_added` ×2 |

Seeded rows have no `payment_method` (`null`). Automatic transactions: **ON** for Mona, **OFF** for Youssef.

Postman: `postman/Darsly-Feature-06-Enrollments-Transactions.postman_collection.json` (folders 06, 07, 11). Have two small jpg / png files and one > 5 MB file ready.

---

## 3. Test cases: record a payment / renewal

As Mona on Omar's Mathematics enrollment unless stated. `POST /teacher/course-enrollments/{id}/transactions`.

| ID | Case | Body | Expected |
|---|---|---|---|
| RN-01 | Amount only | `{amount: 250}` | **201**; `type: renewal`, `"250.00"`, `EGP`; dates unchanged |
| RN-02 | With a method | `{amount: 250, payment_method: "instapay"}` | `payment_method: "instapay"` |
| RN-03 | Each method | `cash`, `vodafone_cash`, `instapay`, `other` | accepted |
| RN-04 | Unknown method | `bitcoin` | 422 (`payment_method`) |
| RN-05 | Method alone | `{payment_method: "cash"}` | 422 `EMPTY_RENEWAL` |
| RN-06 | Message only / zero amount | `{message: "…"}`; `{amount: 0}` | 201; `"0.00"` |
| RN-07 | +30 days | the app sends `subscription_ends_at` = current end + 30 days, with an amount | 201; `changes.subscription_ends_at {from, to}`; the enrollment's end moved; `access_reason` updated |
| RN-08 | Dates only | `{subscription_ends_at: "<future>"}` | 201; renewal without amount |
| RN-09 | Same dates as now | the current end date | 422 `EMPTY_RENEWAL` (not a change) |
| RN-10 | Empty | `{}` | 422 `EMPTY_RENEWAL` |
| RN-11 | Window | start after end | 422 `INVALID_SUBSCRIPTION_DATES` |
| RN-12 | `type: enrollment` | `{type: "enrollment", amount: 100}` | 201; `type: enrollment` |
| RN-13 | Administrative / unknown type | `status_changed`; `payment` | 422 |
| RN-14 | Images | multipart `images[]` (2 files) | 201; 2 `transfer_images` with API URLs |
| RN-15 | Image rules | a pdf; > 5 MB; 11 files | 422 |
| RN-16 | Secretary | Ahmed in Mona's context | 201; `created_by.role: "secretary"` |
| RN-17 | Another teacher's enrollment | Youssef's enrollment from Mona | 403 `ENROLLMENT_NOT_ACCESSIBLE` |
| RN-18 | Never computes dates | renewal with only an amount on an expired enrollment | dates unchanged; still `expired` |

## 4. Test cases: payment summary

`GET /teacher/course-enrollments/{id}` and the list.

| ID | Case | Expected |
|---|---|---|
| PS-01 | Omar (seed) | `total_paid "600.00"`; `last_payment.type "renewal"`, `amount "300.00"`, `payment_method null` |
| PS-02 | After RN-02 | `last_payment` = that renewal (`"250.00"`, `instapay`); `total_paid` increased by 250 |
| PS-03 | Date-only renewal after a payment | `last_payment` does **not** change (no amount) |
| PS-04 | Zero payment after a payment | `last_payment` does not change (`0.00` is skipped); total unchanged |
| PS-05 | Sara (no amount) | `last_payment: null`, `total_paid "0.00"` |
| PS-06 | Laila (0.00) | `last_payment: null`, `total_paid "0.00"` |
| PS-07 | Edit a payment's amount | `total_paid` follows the edited amount |
| PS-08 | Isolation | payments of another student never appear in this summary |
| PS-09 | List vs details | the same values in `GET …?course_id=` and `GET …/{id}` |
| PS-10 | Student app | `GET /student/course-enrollments/{id}` has **no** `last_payment`, `total_paid` or `amount` |
| PS-11 | No secrets | the enrollment never contains transaction messages or transfer images |

## 5. Test cases: history

| ID | Case | Expected |
|---|---|---|
| HI-01 | `GET …/{id}/transactions` | newest first; `created_by` with name and role |
| HI-02 | `?type=renewal` | renewals only; `?type=payment` → 422 |
| HI-03 | `?from=<today>&to=<today>` | today's rows; `to` before `from` → 422 |
| HI-04 | Snapshot | after RN-07, the row shows the **new** dates; older rows keep their old dates |
| HI-05 | `status_changed` | `changes {from, to, reason}`; reasons `explicit`, `last_active_group_removed`, `group_added`, `enrollment_restored` |
| HI-06 | Removed enrollment | delete it (Feature 05); its history and images stay readable |
| HI-07 | One transaction | `GET /teacher/enrollment-transactions/{id}` → 200; another teacher's → 403; unknown → 404 |
| HI-08 | Admin row | a row recorded from the admin panel shows `created_by.by_admin: true` |

## 6. Test cases: edit & images

`PATCH /teacher/enrollment-transactions/{id}` (multipart: `POST` + `_method=PATCH`).

| ID | Case | Body | Expected |
|---|---|---|---|
| ED-01 | Amount + message | `{amount: 200, message: "Corrected"}` | 200; **same id**; `updated_at` moved |
| ED-02 | Method only | `{payment_method: "cash"}` | changed; amount kept |
| ED-03 | Omit method | `{message: "x"}` | method kept |
| ED-04 | Clear | `{payment_method: null}`; `{amount: null}` | `null` |
| ED-05 | Protected fields | `{type, changes, created_by_user_role_id, subscription_ends_at}` | ignored |
| ED-06 | Administrative row | a `group_added` row | 422 `TRANSACTION_NOT_EDITABLE` |
| ED-07 | Add images | `images[]` | added; existing ones kept |
| ED-08 | Remove images | `remove_media_ids: [<own id>]` | removed; file gone after the commit |
| ED-09 | Foreign / unknown media id | an image of another transaction; `999999` | 422 `INVALID_MEDIA_REMOVAL`; nothing changes |
| ED-10 | Download | `GET …/transfer-images/{media}` with headers | 200 file; `Cache-Control: private, no-store`; without the token → 401 |
| ED-11 | Wrong pairing | a media id of another transaction in the URL | 404 `TRANSFER_IMAGE_NOT_FOUND` |
| ED-12 | No delete | `DELETE /teacher/enrollment-transactions/{id}` | 405 |
| ED-13 | Another teacher | edit / view / download a Youssef transaction as Mona | 403 `TRANSACTION_NOT_ACCESSIBLE` |

## 7. Test cases: automatic transactions setting

| ID | Case | Expected |
|---|---|---|
| AT-01 | `GET /teacher/settings` as Mona / as Ahmed in Mona's context | `automatic_transactions: true` |
| AT-02 | As Youssef | `false` |
| AT-03 | Mona (ON): suspend a student | a `status_changed` row |
| AT-04 | Youssef (OFF): attach / detach a group, change the status | applied; **no** `group_*` / `status_changed` rows |
| AT-05 | Youssef (OFF): change the dates | a `renewal` row **is** written |
| AT-06 | Youssef (OFF): new enrollment | only the `enrollment` row |
| AT-07 | Admin turns Youssef ON | the next request writes administrative rows; nothing is back-filled; `GET /teacher/settings` → `true` |
| AT-08 | Read-only | `PATCH /teacher/settings` → 405; Youssef with Mona's context → 403 |

## 8. Security & localization

| ID | Case | Expected |
|---|---|---|
| SC-01 | Unlinked / suspended / expired secretary | 403 with the Feature 2 codes |
| SC-02 | Student token on teacher routes | 403 / context errors |
| SC-03 | Throttle | 121 writes within a minute → 429 with `Retry-After` |
| SC-04 | `Accept-Language: ar` vs `en` on `EMPTY_RENEWAL` | same `error_code`, translated message |

## 9. Admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | *Enrollments → an enrollment → Record transaction* | a renewal on the teacher's behalf; `by_admin: true` in the API |
| AD-02 | *Edit transaction* | amount / message / images change; `payment_method` is kept (the admin form doesn't show it yet) |
| AD-03 | *Users → teacher → Enable / Disable automatic transactions* | audited; changes AT-01 / AT-02 |
| AD-04 | Support admin | read-only |

## 10. Mobile checks

| ID | Check |
|---|---|
| MB-01 | The payment card shows last payment (amount, method, date) and total paid; "No payments yet" when `last_payment` is null. |
| MB-02 | The payment form has an optional method picker and blocks an empty submit. |
| MB-03 | "+30 days" fills the end date in the form; the saved renewal shows the old → new dates. |
| MB-04 | Transfer images load inside the app (with auth) and can't be opened by a shared link. |
| MB-05 | No edit on administrative rows; no delete anywhere. |
| MB-06 | A hint on the history when automatic transactions are off. |

## 11. Bug report

Account, `X-Teacher-Id`, enrollment / transaction / media ids, the request and the full response (`error_code`, `errors`), the test ID and the time. Attach the image file if an upload fails. No tokens or passwords.
