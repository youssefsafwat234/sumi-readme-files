# Feature 14 — Files · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-14-files.md](../features/feature-14-files.md) · mobile guide: [feature-14-files-mobile.md](feature-14-files-mobile.md)

---

## 1. Scope

* Upload (type by content, size), processing (pages, failures), edit, replace, delete, purge.
* Groups, category, lecture link, publish date, protection flags.
* Student list, access (signed URL, watermark, flags), lock through the lecture, open counts.
* Admin panel: Courses → a course → *Files* tab.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
php artisan queue:work    # processing (or QUEUE_CONNECTION=sync)
```

| Seeded file | Pages | Attached to | Category |
|---|---|---|---|
| Preface | 8 | Preface lecture | — |
| Chapter 1 · Part 1 | 14 | its lecture | — |
| Chapter 1 · Part 1 — Solutions | 6 | — | — |
| Section 1 & 2 | 22 | — | «أخرى» |

Upload fixtures: `postman/fixtures/` (`summary-8-pages.pdf`, `summary-14-pages.pdf`, `corrupt.pdf`, `encrypted.pdf`, `program-renamed.pdf`, `notes.txt`). Postman: `postman/Darsly-Feature-14-Files.postman_collection.json`.

---

## 3. Test cases: upload & processing

| ID | Case | Expected |
|---|---|---|
| FU-01 | 8-page PDF | 201, `status: processing` → `ready`, `pages: 8` |
| FU-02 | JPG / PNG / WEBP | `ready`, `pages: 1` |
| FU-03 | `program-renamed.pdf` (an exe renamed) | 422 `FILE_TYPE_NOT_SUPPORTED`, `details.allowed_types` |
| FU-04 | `notes.txt`, a .docx | 422 `FILE_TYPE_NOT_SUPPORTED` |
| FU-05 | 60 MB file | 422 `FILE_TOO_LARGE` (`details.max_mb: 50`) |
| FU-06 | `corrupt.pdf` / `encrypted.pdf` | `failed` with `corrupt` / `encrypted` |
| FU-07 | PDF with JavaScript | `failed: unsafe_content` |
| FU-08 | No groups / group of another course | 422 / 422 `INVALID_GROUP_ASSIGNMENT` |
| FU-09 | `lecture_id` of another course | 422 `INVALID_LECTURE` |
| FU-10 | Flags omitted | defaults: watermark ✓, highlights ✓, download ✗, print ✗ |
| FU-11 | 21 uploads in a minute | 429 |

## 4. Test cases: edit, replace, delete

| ID | Case | Expected |
|---|---|---|
| FE-01 | PATCH title / flags | changed; file unchanged |
| FE-02 | PATCH with `file` | 422 (use replace) |
| FE-03 | Replace | same id; new metadata; a student URL issued before → 404 |
| FE-04 | Delete | gone from lists; student access 404 |
| FE-05 | Filters `?status=failed`, `?lecture_id=`, `?search=` | filtered |
| FE-06 | Group tab | only files of that group |

## 5. Test cases: student

| ID | Case | Expected |
|---|---|---|
| FS-01 | Nour lists math files | ready + published + her groups only; no URL in the list |
| FS-02 | Draft / scheduled / failed / processing file | not listed; access → 404 / 409 `FILE_NOT_READY` |
| FS-03 | Access | `url` + `expires_at` (5 min), flags, `watermark` |
| FS-04 | Open the URL without a token | file streams inline, `Cache-Control: private, no-store` |
| FS-05 | Open the URL with **another** user's token | 403 `FILE_NOT_ACCESSIBLE` |
| FS-06 | URL after 5 minutes / tampered signature | 403 |
| FS-07 | File of a lecture locked for Nour (Chapter 2 · Part 1) | `locked: true`; access → 403 `FILE_LOCKED` |
| FS-08 | Watermark off | `watermark: null` |
| FS-09 | Open 3 times in 10 minutes | `opens_count` +1 only |
| FS-10 | Lecture view start | lists the lecture's ready files under `files` |
| FS-11 | Course access window closed (Feature 22) | 403 `COURSE_NOT_OPEN_YET` / `COURSE_ACCESS_CLOSED` |

## 6. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Files tab | type, size, pages, status, groups, opens |
| AD-02 | Upload / edit / replace / delete | same rules |

## 7. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Watermark visible on every page. |
| MB-02 | Print button hidden / «الطباعة مقفولة» when `allow_print: false`. |
| MB-03 | Download for offline only when `allow_download: true`. |
| MB-04 | Expired link silently re-requested. |

## 8. Bug report

File / course / group / lecture ids, the uploaded file (name, size, type), request and full response (`error_code`, `details`), the test ID and the time.


### Teacher files list response

`GET /teacher/files` returns file metadata and pagination without the `category`, `lecture`, or `groups` objects. These keys are omitted, not null. Query filters for these relations remain available. Course-scoped lists and file details retain their existing relation data. Verify the three keys are absent even when the file has assigned relations.
