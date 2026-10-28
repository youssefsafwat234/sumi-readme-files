# Feature 18 — Exam attempts & results · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-18-exam-attempts-results.md](../features/feature-18-exam-attempts-results.md) · mobile guide: [feature-18-exam-attempts-results-mobile.md](feature-18-exam-attempts-results-mobile.md)

---

## 1. Scope

* Start rules, resume, order (shuffle), answers and flags, time limit and automatic submit.
* Grading (snapshot, exact match, no partial credit), pass / fail, best attempt.
* Result visibility matrix, review, rank.
* Teacher results, student attempts, attempt detail.
* Links: lecture conditions (Feature 11), exam question lock (Feature 16), progress (Feature 19).

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

On «امتحان الوحدة الأولى» (20 questions, 30 min): **Omar** 14 / 20 passed, **Laila** 6 / 20 failed, **Nour** attempt in progress with 5 answers. The automatic submit runs every minute (`php artisan schedule:work`) or by hand: `php artisan darsly:exams:submit-expired`.

Postman: `postman/Darsly-Feature-18-Exam-Attempts.postman_collection.json`.

---

## 3. Test cases: start

| ID | Case | Expected |
|---|---|---|
| AS-01 | Info as Sara (not attempted) | `can_start: true`, attempts `{used: 0, allowed, remaining}` |
| AS-02 | Start | **201**; questions with labels; no `is_correct` anywhere |
| AS-03 | Start again while in progress | **200**, same attempt, same order, saved answers |
| AS-04 | Nour (in progress) info | `in_progress_attempt_id` set |
| AS-05 | Scheduled exam / closed exam | 403 `EXAM_NOT_OPEN` / `EXAM_CLOSED` |
| AS-06 | All attempts used | 403 `EXAM_NO_ATTEMPTS_LEFT` |
| AS-07 | Draft exam / not in the exam's groups | 404 `EXAM_NOT_FOUND` |
| AS-08 | Two starts at once | one attempt |
| AS-09 | Shuffled exam | order differs between students; stable per attempt; صح / خطأ keep their order |
| AS-10 | `closes_at` in 10 min, duration 30 | `deadline_at` = `closes_at` |

## 4. Test cases: answers & submit

| ID | Case | Expected |
|---|---|---|
| AA-01 | Single choice, one id | saved |
| AA-02 | Single choice, two ids / an id of another question | 422 `INVALID_ANSWER` |
| AA-03 | Multiple choice, two ids | saved |
| AA-04 | `[]` | answer cleared |
| AA-05 | `flagged: true` alone | flag saved; progress `flagged` +1 |
| AA-06 | Answer after the deadline | 409 `ATTEMPT_TIME_UP`; attempt finalised |
| AA-07 | Submit twice | same result (idempotent) |
| AA-08 | Answer after submit | 409 `ATTEMPT_ALREADY_SUBMITTED` |
| AA-09 | Expired attempt, run `darsly:exams:submit-expired` | submitted with `time_up`; `submitted_at` = the deadline |
| AA-10 | Read an expired attempt before the command runs | finalised on read |

## 5. Test cases: grading & visibility

| ID | Case | Expected |
|---|---|---|
| AG-01 | Multiple choice, one of two correct | wrong, 0 points |
| AG-02 | Unanswered | wrong |
| AG-03 | Teacher edits a question after the attempt started | the attempt keeps the original text / options / images |
| AG-04 | `immediately` + correct answers | score, `correct_option_ids`, explanation |
| AG-05 | `immediately`, no correct answers | `is_correct` only |
| AG-06 | `after_close` before / after close | pending with `available_at` / result |
| AG-07 | `hidden` | `result_visible: false` |
| AG-08 | Two attempts 60 % and 40 % | counted = 60 % (best) |
| AG-09 | Passing an exam required by a lecture | the lecture opens (Feature 11) |
| AG-10 | Rank | `rank.out_of` = students who submitted |

## 6. Test cases: teacher

| ID | Case | Expected |
|---|---|---|
| AT-01 | Results of unit 1 | Omar 70 % passed, Laila 30 % failed; summary |
| AT-02 | `?passed=0` / `?group_id=` | filtered |
| AT-03 | Omar's attempts | list with counts |
| AT-04 | Attempt detail | snapshots with correct answers |
| AT-05 | Another teacher's attempt | 404 `ATTEMPT_NOT_FOUND` |
| AT-06 | Exam with an attempt: edit questions | 409 (Feature 16) |
| AT-07 | Exam card stats | `attempted_students`, `average_percentage` |

## 7. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Timer correct when the phone clock is wrong (uses `server_time`). |
| MB-02 | Killing the app and reopening resumes the attempt. |
| MB-03 | Result screen for each visibility setting. |
| MB-04 | Offline answer retries without losing the selection. |

## 8. Bug report

Student, exam / attempt / question ids, request and full response, `server_time` and `deadline_at`, the test ID and the time.
