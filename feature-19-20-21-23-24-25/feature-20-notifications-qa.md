# Feature 20 — Course notifications & push · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-20-notifications.md](../features/feature-20-notifications.md) · mobile guide: [feature-20-notifications-mobile.md](feature-20-notifications-mobile.md)

---

## 1. Scope

* Compose (title, body, audience), preview, send (async), retry, delete, history with read stats, rate limit.
* Recipients rule (access now, once per student), student inbox, badge, read / read all.
* Firebase: topic push (course / groups), token push (> 5 groups), private message and subscription reminder pushes, tokens on sign-in / refresh / logout, topic sync.
* Admin panel: Courses → a course → *Notifications* tab.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
php artisan queue:work
```

* Demo course: «تأجيل محاضرة السبت» (to group A) and «ملخص المحاضرة التالتة» (all, read by all but one), plus 10 older announcements (12 in total).
* `DARSLY_PUSH_DRIVER=log` (default): every push / topic subscription is written to `storage/logs/laravel.log` (`fcm.topic`, `fcm.condition`, `fcm.tokens`, `fcm.subscribe`, `fcm.unsubscribe`). For real phones set `DARSLY_PUSH_DRIVER=fcm` + the service account (see the feature doc §4).

Postman: `postman/Darsly-Feature-20-Notifications.postman_collection.json` (folder 10: Firebase).

---

## 3. Test cases: compose & send

| ID | Case | Expected |
|---|---|---|
| NS-01 | Preview `all` | `recipients_count` = students with access; suspended / expired excluded |
| NS-02 | Preview two groups with a shared student | counted once in the total; in both `per_group` |
| NS-03 | Send to all | 201 `sending` → `sent`; recipients stored |
| NS-04 | `group_ids` with `all` / `groups` without ids | 422 |
| NS-05 | Group of another course | 422 `INVALID_GROUP_ASSIGNMENT` |
| NS-06 | Audience with nobody | 422 `NOTIFICATION_NO_RECIPIENTS` |
| NS-07 | 11th send in an hour | 429 `NOTIFICATION_RATE_LIMITED` (`Retry-After`) |
| NS-08 | Title 81 / body 1001 characters | 422 |
| NS-09 | Retry a `sent` one | 409 `NOTIFICATION_NOT_RETRYABLE` |
| NS-10 | Secretary sends | `sent_by.role: secretary` |
| NS-11 | Delete | gone from history and inbox |
| NS-12 | Student joining after the send | doesn't see it |

## 4. Test cases: student

| ID | Case | Expected |
|---|---|---|
| NI-01 | Inbox | newest first, course and sender |
| NI-02 | Badge | `unread_count` |
| NI-03 | Read | `read_at` set; teacher `read_count` +1 |
| NI-04 | Read again | idempotent; count unchanged |
| NI-05 | Read all (`?course_id=`) | only that course |
| NI-06 | A notification not sent to this student | 404 |

## 5. Test cases: Firebase

| ID | Case | Expected (log driver: check the log) |
|---|---|---|
| NF-01 | Login with `fcm_token` + `platform` | token stored; `fcm.subscribe` for `darsly_course_{id}` and `darsly_group_{id}` of the student's groups |
| NF-02 | `fcm_token` without `platform` | 422 |
| NF-03 | Login on a second device | the first token deleted and unsubscribed |
| NF-04 | `POST /me/push-tokens` (refresh) | stored and subscribed |
| NF-05 | Logout | token deleted; `fcm.unsubscribe` |
| NF-06 | Send to all | **one** `fcm.topic` push (course topic) |
| NF-07 | Send to 2 groups | one `fcm.condition` push (`'…group_a' in topics || '…group_b' in topics`) |
| NF-08 | Send to 6 groups | `fcm.tokens` push to the recipients' tokens |
| NF-09 | Remove a student from a group / suspend the enrollment | the token leaves that topic |
| NF-10 | `php artisan darsly:push:sync-topics` | subscribes / unsubscribes the differences |
| NF-11 | Private message student → teacher | `fcm.tokens` to the teacher and active secretaries; `data.type: private_message` |
| NF-12 | Subscription reminder (Feature 23, 7 / 3 / 1 days) | push to the teacher; `data.type: platform_subscription` |
| NF-13 | (fcm driver) dead token | deleted; `push_error` stored on the recipient; the notification stays `sent` |

## 6. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Notifications tab | history with recipients, reads, status |

## 7. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Push received in foreground / background / killed state. |
| MB-02 | Tap opens the right screen per `data.type`. |
| MB-03 | After logout the phone no longer receives pushes. |
| MB-04 | Signing in on another phone stops pushes on the first. |

## 8. Bug report

Course / notification ids, the request and full response, the log lines (`fcm.*`) or the Firebase error, device platform, the test ID and the time.
