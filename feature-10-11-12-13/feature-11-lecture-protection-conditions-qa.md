# Feature 11 — Lecture protection & opening conditions · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-11-lecture-protection-conditions.md](../features/feature-11-lecture-protection-conditions.md) · mobile guide: [feature-11-lecture-protection-conditions-mobile.md](feature-11-lecture-protection-conditions-mobile.md)

---

## 1. Scope

* Protection settings, opening conditions (lectures, exams, pass mark, no loops), the exam before (must pass) / after the lecture.
* Blocked students, extra views, mark watched.
* Student lock states, starting a view (view counting), heartbeats and "watched".
* Admin panel row actions on lectures.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

| Seeded | Setting |
|---|---|
| Preface | unlimited views |
| other lectures | 50 views |
| Production practical | requires headphones; **Laila is blocked** |
| Chapter 2 · Part 1 | requires both Chapter 1 parts — **open for Omar** (watched both), **locked for Nour** (watched Part 1 only) |

Postman: `postman/Darsly-Feature-11-Lecture-Protection.postman_collection.json` (folder 10: lecture exams).

---

## 3. Test cases: protection

| ID | Case | Expected |
|---|---|---|
| PT-01 | GET on a new lecture | defaults: `max_views: null`, screenshots false, offline false, watermark true, headphones false |
| PT-02 | PATCH `{allow_screenshots: true}` | only that field changes |
| PT-03 | `max_views: 0` / 1001 | 422 |
| PT-04 | Lower `max_views` below views used | 200; `views_used` kept; student now exhausted |
| PT-05 | `blocked_students: [X]` then `[]` | X blocked, then nobody |
| PT-06 | Block an enrollment of another course | 422 `STUDENT_NOT_IN_COURSE` |

## 4. Test cases: conditions

| ID | Case | Expected |
|---|---|---|
| CN-01 | Require Chapter 1 Part 1 | saved; GET lists it with title and number |
| CN-02 | Require itself | 422 `INVALID_REQUIRED_LECTURE` |
| CN-03 | A → B then B → A | 422 `CONDITION_CYCLE` |
| CN-04 | A lecture of another course / deleted | 422 `INVALID_REQUIRED_LECTURE` |
| CN-05 | A draft exam / unknown exam | 422 `INVALID_EXAM` |
| CN-06 | A published exam + `min_pass_percentage: 60` | saved; locked until 60 % |
| CN-07 | `min_pass_percentage: 101` | 422 |
| CN-08 | PUT `{}` | all conditions removed |

## 5. Test cases: exam before / after

| ID | Case | Expected |
|---|---|---|
| PE-01 | Create a lecture with `pre_exam_id` + `pre_exam_must_pass: 1` + `post_exam_id` | 201; detail `pre_exam {must_pass: true}`, `post_exam`; the exam's `lecture_id` = the lecture |
| PE-02 | `pre_exam_must_pass: 1` without an exam | 422 `VALIDATION_ERROR` |
| PE-03 | Same exam before and after | 422 |
| PE-04 | Exam of another course | 422 `INVALID_EXAM` |
| PE-05 | Draft exam | accepted |
| PE-06 | Student who hasn't passed it | locked, `conditions[0].type: pre_exam`, `required` = exam pass mark |
| PE-07 | Student passes it | lecture opens |
| PE-08 | `pre_exam_must_pass: 0` | not locked; `pre_exam` still shown |
| PE-09 | Change `post_exam_id` | old exam unlinked, new linked; `null` unlinks |

## 6. Test cases: student playback

| ID | Case | Expected |
|---|---|---|
| PB-01 | Omar opens Chapter 2 · Part 1 | `can_open: true` |
| PB-02 | Nour opens it | locked `conditions_not_met`; `POST views` → 403 `LECTURE_LOCKED` with `details.conditions` |
| PB-03 | Teacher marks Part 2 watched for Nour | it opens |
| PB-04 | Laila opens Production practical | 403 `LECTURE_BLOCKED` |
| PB-05 | Views limit 2: start 3 views | 3rd → 403 `LECTURE_VIEWS_EXHAUSTED` |
| PB-06 | Grant 1 extra view | the 3rd view works |
| PB-07 | Two devices start the last view at once | only one succeeds |
| PB-08 | Watermark flag on / off | `watermark {name, code}` / `null` |
| PB-09 | Heartbeat twice within 10 s | 2nd → 429 |
| PB-10 | Heartbeat with a 600 s delta after 10 s | counted as at most the real elapsed time + 5 s |
| PB-11 | Heartbeat on another student's view | 404 `VIEW_NOT_FOUND` |
| PB-12 | Watch ≥ 90 % | `completed: true`; lectures requiring it open |
| PB-13 | Block the student mid-playback | next heartbeat 403 `LECTURE_BLOCKED` |

## 7. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Lecture row → Protection settings / Opening conditions | same rules as the API |
| AD-02 | Block / unblock / extra views / mark watched | applied; audit actor = admin |

## 8. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Screenshots blocked when `allow_screenshots: false`. |
| MB-02 | Playback pauses without headphones when required. |
| MB-03 | Watermark visible and moving over the video. |
| MB-04 | Locked lecture shows the checklist; tapping the pre-exam opens the exam. |

## 9. Bug report

Account, lecture / enrollment / view ids, request and full response (`error_code`, `details`), the test ID and the time.
