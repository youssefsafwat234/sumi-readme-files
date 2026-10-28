# Feature 15 — Question bank · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-15-question-bank.md](../features/feature-15-question-bank.md) · mobile guide: [feature-15-question-bank-mobile.md](feature-15-question-bank-mobile.md)

---

## 1. Scope

* Bank list: filters, search (body + options), type counts, sort, exam picker modes.
* Create / edit (option rules per type), copy, delete from the bank (usage kept).
* Question images: add, delete, replace, crop, reorder, limits.
* Usage shown on edit; images shown in exams, attempts and videos.
* Admin panel: Courses → a course → *Questions* tab.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

Mona's math course has **128 questions**: 41 true / false, 62 single choice, 25 multiple choice, spread over «الوحدة الأولى / الثانية / التالتة». Image fixture: `postman/fixtures/message-photo.jpg` (320 × 200).

Postman: `postman/Darsly-Feature-15-Question-Bank.postman_collection.json` (folder 07: images).

---

## 3. Test cases: list

| ID | Case | Expected |
|---|---|---|
| QL-01 | No filter | 15 per page; `meta.counts_by_type {all: 128, true_false: 41, single_choice: 62, multiple_choice: 25}` |
| QL-02 | `type[]=true_false&type[]=single_choice` / `type=true_false,single_choice` | 103 results; counts unchanged |
| QL-03 | `category_id` / `uncategorized=1` / both | filtered / filtered / 422 |
| QL-04 | `search` matching only an option text | the question is found |
| QL-05 | `not_in_exam={exam}` | questions of that exam hidden |
| QL-06 | `exam_id={exam}` | `in_exam` true for its questions |
| QL-07 | `sort=oldest` | oldest first |
| QL-08 | `/teacher/questions` without course | every course of Mona, each with `course_id` |
| QL-09 | `type=essay` | 422 |

## 4. Test cases: create, edit, copy, delete

| ID | Case | Expected |
|---|---|---|
| QC-01 | Each type with valid options | 201 |
| QC-02 | true/false with 3 options; single with 2 correct; multiple with 0 correct; 9 options; duplicate option texts | 422 `INVALID_QUESTION_OPTIONS` |
| QC-03 | Change only `type` to true_false on a 4-option question | 422 (stored options re-checked) |
| QC-04 | `add_to_exam_id` | created and added to that exam |
| QC-05 | Edit a question used in 2 exams | 200; response `usage.exams: 2`, `exams[]` listed |
| QC-06 | Copy | 201 new id, `usage.total: 0`, images copied |
| QC-07 | Delete a used question | 200 `{usage}`; gone from the bank; the exams still show it |
| QC-08 | Category of another course | 422 `INVALID_CATEGORY` |

## 5. Test cases: images

| ID | Case | Expected |
|---|---|---|
| QI-01 | Create with 2 images (multipart) | `images[2]` with `url` / `thumb_url`, upload order |
| QI-02 | 6 images on create | 422 |
| QI-03 | A pdf as an image | 422 `VALIDATION_ERROR` |
| QI-04 | 4 images, then add 2 | 422 `QUESTION_IMAGES_LIMIT` |
| QI-05 | Remove one (`remove_image_ids[]`) | removed |
| QI-06 | Remove an image of another question | 422 `QUESTION_IMAGE_NOT_FOUND` |
| QI-07 | Replace image 2 | new id at position 2 |
| QI-08 | Crop 300 × 200 at (100, 50) | new id; stored image is 300 × 200 |
| QI-09 | Crop outside the image | 422 `INVALID_IMAGE_CROP`; nothing changed |
| QI-10 | Reorder (JSON `image_order`) | new order; missing an image → 422 `INVALID_IMAGE_ORDER` |
| QI-11 | Same image removed and cropped | 422 `INVALID_IMAGE_OPERATIONS` |
| QI-12 | All together + a body change | applied in one request; order remove → replace → crop → reorder → add |
| QI-13 | Bank card | `images_count`, `image_thumb_url` |
| QI-14 | Exam detail / student attempt / video timeline | `images[]` shown; the attempt keeps the images it started with |

## 6. Test cases: access

| ID | Case | Expected |
|---|---|---|
| QA-01 | Ahmed (secretary) creates | 201 |
| QA-02 | Youssef on Mona's course | 403 `COURSE_NOT_ACCESSIBLE` |
| QA-03 | A question of another course in this URL | 404 `QUESTION_NOT_FOUND` |

## 7. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Questions tab | filters by type / category; copy and delete actions |
| AD-02 | Create / edit with the options repeater | same option rules |

## 8. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Edit warning shows the usage count. |
| MB-02 | Image viewer actions (delete, replace, crop, reorder) save together. |
| MB-03 | Crop coordinates are scaled to the stored image. |

## 9. Bug report

Course / question / image ids, request (multipart keys) and full response (`error_code`, `details`), the test ID and the time.
