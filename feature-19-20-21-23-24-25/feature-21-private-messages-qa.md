# Feature 21 — Private messages · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-21-private-messages.md](../features/feature-21-private-messages.md) · mobile guide: [feature-21-private-messages-mobile.md](feature-21-private-messages-mobile.md)

---

## 1. Scope

* Course messaging settings and their link to the Feature 08 permission.
* Send rules per side, kinds (text / image / voice), media validation, signed media links.
* Threads, paging cursors, read state and unread counts, delete window.
* Push to the other side (Feature 20).
* Live updates over Pusher (`message.sent`, `message.deleted`, `thread.read`).
* Admin panel: Courses → view → «Private messages» action.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

* Messages are **on** for the demo course.
* Omar's thread: his question, the secretary's reply, a voice note (18 s). Laila: an empty thread.
* Fixtures: `postman/fixtures/message-photo.jpg`, `voice-note.mp3`.

Postman: `postman/Darsly-Feature-21-Private-Messages.postman_collection.json`.

---

## 3. Test cases: settings & send rules

| ID | Case | Expected |
|---|---|---|
| MS-01 | Messages off, student sends | 403 `MESSAGES_DISABLED`; thread still readable, `reason: disabled` |
| MS-02 | Student permission off (Feature 08 override) | 403 `MESSAGES_BLOCKED_FOR_STUDENT` |
| MS-03 | Override **on** while the course is off | still 403 `MESSAGES_DISABLED` |
| MS-04 | Expired subscription, `allow_non_subscribed` off / on | 403 `MESSAGES_SUBSCRIPTION_REQUIRED` / 201 |
| MS-05 | `allow_images` off, image | 403 `MESSAGE_KIND_NOT_ALLOWED` |
| MS-06 | Staff sends with messages off | 201 |
| MS-07 | Staff opens a student with no thread (`by-enrollment`) | thread created, empty (L15.4) |
| MS-08 | Settings change | audit `course.messaging_settings_updated` |

## 4. Test cases: kinds & media

| ID | Case | Expected |
|---|---|---|
| MK-01 | Text 2000 / 2001 characters | 201 / 422 |
| MK-02 | Image jpg | 201, `media.url` + `thumb_url` |
| MK-03 | A pdf renamed `.jpg` | 422 `INVALID_MEDIA` (`type`) |
| MK-04 | Image 11 MB | 422 `INVALID_MEDIA` (`size`) |
| MK-05 | Voice without `duration_seconds` / 301 | 422 `INVALID_MEDIA` (`duration`) |
| MK-06 | Media link without a token | streams |
| MK-07 | Media link with **another** user's token | 404 `MESSAGE_NOT_FOUND` |
| MK-08 | Media link after 5 minutes | refused |

## 5. Test cases: threads, read, delete

| ID | Case | Expected |
|---|---|---|
| MT-01 | Staff thread list | only threads with messages, newest first, `unread_total` |
| MT-02 | `?unread=1`, `?search=Omar` | filtered |
| MT-03 | `?before=` / `?after=` | older / newer pages, oldest → newest; `has_more` |
| MT-04 | Student opens the thread (`read`) | staff messages get `read_at`; student counter 0 |
| MT-05 | Student unread badge | across all courses |
| MT-06 | Student deletes own message at 10 min / 20 min | 200 / 409 `MESSAGE_DELETE_WINDOW_PASSED` |
| MT-07 | Staff deletes the student's message | 200; shown as `deleted: true` |
| MT-08 | Another student's thread / a teacher of another course | 404 / 403 |
| MT-09 | Unlinked secretary | 403 |

## 6. Test cases: realtime (Pusher)

Use the Pusher dashboard **Debug console** (app of `PUSHER_APP_ID`) to watch events, and `POST /api/v1/broadcasting/auth` (form `socket_id=1234.5678`, `channel_name=…`) for the authorization.

| ID | Steps | Expected |
|---|---|---|
| RT-01 | Student / teacher / active secretary authorize `private-thread.{id}` | 200 `{auth}` |
| RT-02 | Another student, another teacher, a secretary whose link is off | 403 |
| RT-03 | No bearer | 401 |
| RT-04 | `private-user.{ownId}` vs someone else's id | 200 / 403 |
| RT-05 | Send a message | `message.sent` on the thread channel and on each participant's user channel; no media URL in the payload |
| RT-06 | Delete a message | `message.deleted` |
| RT-07 | Open the thread with unread messages, then again | `thread.read` once (nothing marked the second time) |

## 6.1 Test cases: push

| ID | Case | Expected |
|---|---|---|
| MP-01 | Student sends | push to the teacher and active secretaries (`data.type: private_message`), not to the student |
| MP-02 | Secretary replies | push to the student only |
| MP-03 | Image / voice | push body «بعت صورة.» / «بعت رسالة صوتية.» |

## 7. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Course → «Private messages» | the four switches saved; no message content visible anywhere in the panel |

## 8. Mobile checks

| ID | Check |
|---|---|
| MB-01 | «بواسطة السكرتير» on secretary messages. |
| MB-02 | Composer hidden with the right reason. |
| MB-03 | Voice note plays with its duration. |
| MB-04 | Push opens the thread. |

## 9. Bug report

Course / thread / message ids, sender side, request (multipart keys) and full response (`error_code`, `details`), the test ID and the time.
