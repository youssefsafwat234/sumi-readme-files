# Feature 16 — Exams · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-16-exams.md](../features/feature-16-exams.md) · mobile guide: [feature-16-exams-mobile.md](feature-16-exams-mobile.md)

---

## 1. Scope

* Exam drafts, settings, questions (add, points, remove, reorder), totals.
* Publish checklist, publish on create, unpublish, delete, derived status.
* Question lock after the first attempt.
* Lists (course, group tab), student list and visibility.
* Admin panel: Courses → a course → *Exams* tab.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

On Mona's math course, every group:

| Exam | Setup |
|---|---|
| «امتحان الوحدة الأولى» | 20 questions, 30 min, 20 points, **open** |
| «امتحان الوحدة الثانية» | 12 questions, quiz, 1 attempt, **scheduled** (next Tuesday 18:00 → Thursday 23:59) |
| «Final · مراجعة شاملة» | 25 questions, final, 40 min, 1 attempt |
| «امتحان الوحدة التالتة» | **draft**, no questions |

Postman: `postman/Darsly-Feature-16-Exams.postman_collection.json`.

---

## 3. Test cases: create & edit

| ID | Case | Body | Expected |
|---|---|---|---|
| EC-01 | Empty draft | `{}` | 201; `status: draft`; number = next; defaults |
| EC-02 | Every field | full body | 201; totals = sum of effective points |
| EC-03 | Number taken | existing number | 422 `EXAM_NUMBER_TAKEN` |
| EC-04 | `closes_at` before `opens_at` | | 422 `INVALID_EXAM_WINDOW` |
| EC-05 | Question of another course / deleted | | 422 `INVALID_QUESTION` |
| EC-06 | 201 questions | | 422 |
| EC-07 | `publish: true` with nothing else | | 422 `EXAM_NOT_PUBLISHABLE`, `missing [title, questions, groups]`; **no exam created** |
| EC-08 | `publish: true` complete | | 201 "Exam published successfully."; `status: open` |
| EC-09 | Edit `group_ids` / `questions` | | replaced |
| EC-10 | `max_attempts: null` | | unlimited |
| EC-11 | Lecture of another course in `lecture_id` | | 422 `INVALID_LECTURE` |

## 4. Test cases: questions

| ID | Case | Expected |
|---|---|---|
| EQ-01 | Add 2 questions with `points: 2` | appended; totals updated |
| EQ-02 | Add a question already in the exam | no duplicate |
| EQ-03 | Points `null` | uses the question's default |
| EQ-04 | Reorder with exactly the exam's questions / missing one | 200 / 422 |
| EQ-05 | Remove | gone from the exam; still in the bank |
| EQ-06 | Create a question with `add_to_exam_id` | added; invalid exam → nothing created |
| EQ-07 | After a student attempt: add / remove / reorder / change questions | 409 `EXAM_HAS_ATTEMPTS` |
| EQ-08 | After an attempt: change duration / visibility | 200 |
| EQ-09 | Same `questions` list unchanged after attempts | 200 (not a change) |

## 5. Test cases: publish & status

| ID | Case | Expected |
|---|---|---|
| EP-01 | Publish the draft unit-3 exam | 422, `missing [questions, groups]` (title present) |
| EP-02 | Detail checklist | `ready`, `missing[]`, `items[{key, ok}]` match |
| EP-03 | Publish complete, no dates | `open` |
| EP-04 | Future `opens_at` | `scheduled` until then |
| EP-05 | `closes_at` passed | `closed` |
| EP-06 | Published exam: remove all groups / last question / title | 422 `EXAM_NOT_PUBLISHABLE` |
| EP-07 | Unpublish without attempts / with attempts | `draft` / 409 |
| EP-08 | Delete | gone; attempts kept |

## 6. Test cases: lists & student

| ID | Case | Expected |
|---|---|---|
| EL-01 | Course list filters `status`, `type`, `search`, `category_id` | filtered; ordered by number |
| EL-02 | Group tab | that group's exams + `tabs` |
| ES-01 | Nour's exam list | open first, scheduled, closed; `section` set; no questions |
| ES-02 | Draft exam / group she isn't in | not listed |
| ES-03 | Suspended enrollment / course access window closed | not listed / 403 (Feature 22) |

## 7. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Exams tab | status, totals, groups, shared flag |
| AD-02 | Create with the questions repeater, publish, back to draft, delete | same rules |

## 8. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Red dots follow `publish_checklist`. |
| MB-02 | Locked questions screen after attempts. |
| MB-03 | Option labels (أ ب ج / A B C / 1 2 3) shown as chosen. |

## 9. Bug report

Course / exam / question ids, request and full response (`error_code`, `details`), the test ID and the time (status depends on the clock).


### Move one exam question

`PUT /teacher/courses/{course}/exams/{exam}/questions/order`

```json
{"question_id": "{questionId}", "position": 1}
```

Use the bank question ID returned as `questions[].id`, not the exam slot ID. Position starts at 1 and cannot exceed the number of questions in the exam. Other questions shift automatically, preserving relative order. The response is the updated exam with ordered questions. Invalid IDs, questions outside this exam, or invalid positions return 422 without changes. Existing attempts still prevent reordering (409 EXAM_HAS_ATTEMPTS).

Legacy full-list `question_ids` remains supported; do not combine formats. Verify moves to first/last, invalid positions, foreign questions, and the attempt lock.
