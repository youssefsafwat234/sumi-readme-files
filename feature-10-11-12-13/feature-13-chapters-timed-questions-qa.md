# Feature 13 — Video chapters & timed questions · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-13-chapters-timed-questions.md](../features/feature-13-chapters-timed-questions.md) · mobile guide: [feature-13-chapters-timed-questions-mobile.md](feature-13-chapters-timed-questions-mobile.md)

---

## 1. Scope

* Chapters per video (replace), timed questions (existing or inline question), flags and points.
* Student timeline (answers hidden), answering (first answer counts), grading per type.
* Nested chapters / questions in the lecture body.
* Admin panel: Questions tab, «Video chapters», «Add / remove a timed question».

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

First part of «Chapter 1 · Part 2»: chapters `00:00 مقدمة, 04:30 Interest, 11:20 Needs vs Wants, 19:05 Marketing research, 28:30 Evaluation of choices, 36:40 Purchase`; timed questions at **15:00** (single choice) and **31:10** (true / false → False, explanation, 2 points).

Postman: `postman/Darsly-Feature-13-Chapters-Timed-Questions.postman_collection.json`.

---

## 3. Test cases: chapters

| ID | Case | Expected |
|---|---|---|
| CH-01 | PUT 3 chapters out of order | 200; returned sorted |
| CH-02 | Two at the same second | 422 (`chapters.N.start_second`) |
| CH-03 | `start_second` ≥ duration | 422 `CHAPTER_OUT_OF_RANGE` |
| CH-04 | First chapter at 120 s | allowed |
| CH-05 | `[]` | all removed |
| CH-06 | 101 chapters / title 101 chars | 422 |

## 4. Test cases: timed questions

| ID | Case | Expected |
|---|---|---|
| TQ-01 | Add with `question_id` | 201; flags default true |
| TQ-02 | Add with an inline question | 201; the question also appears in the bank |
| TQ-03 | Inline question with bad options (true/false with 3 options) | 422 `INVALID_QUESTION_OPTIONS` (`question.options`); nothing created |
| TQ-04 | Same second twice | 422 `TIMED_QUESTION_TIME_TAKEN` |
| TQ-05 | Delete it, re-add at the same second | 201 |
| TQ-06 | `at_second` ≥ duration | 422 `TIMED_QUESTION_OUT_OF_RANGE` |
| TQ-07 | 31st question | 422 `TIMED_QUESTION_LIMIT_REACHED` |
| TQ-08 | A question of another course | 422 `QUESTION_NOT_FOUND` |
| TQ-09 | `points: null` | uses the question's `default_points` |
| TQ-10 | GET questions after answers | `stats {answered, correct_rate}` |
| TQ-11 | Lecture body nested questions with a time clash | 422 at `videos.0.questions.1.at_second`; lecture unchanged |

## 5. Test cases: student

| ID | Case | Expected |
|---|---|---|
| TS-01 | Timeline | chapters + questions; **no** `is_correct` / correct ids anywhere |
| TS-02 | Answer the single choice correctly | 201, `is_correct: true`, points awarded |
| TS-03 | Answer again (different option) | 200 `already_answered: true`, the first result |
| TS-04 | Multiple choice: one of two correct options | `is_correct: false` (no partial credit) |
| TS-05 | Two options on a single choice / an option of another question | 422 `INVALID_ANSWER` |
| TS-06 | `show_explanation: false` | `explanation: null` |
| TS-07 | Another student's view id | 404 `VIEW_NOT_FOUND` |
| TS-08 | A video of another lecture | 404 `VIDEO_NOT_FOUND` |
| TS-09 | Block the student after the view started | 403 `LECTURE_BLOCKED` |
| TS-10 | Question with images (Feature 15) | `question.images[]` in the timeline |

## 6. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Questions tab | create / edit with options repeater |
| AD-02 | Video chapters / Add / Remove timed question | same rules as the API |

## 7. Mobile checks

| ID | Check |
|---|---|
| MB-01 | The video pauses at the question time and resumes after answering. |
| MB-02 | `must_answer` blocks skipping. |
| MB-03 | Chapter list seeks correctly. |

## 8. Bug report

Lecture / video / view / timed question ids, request and full response, the test ID and the time.
