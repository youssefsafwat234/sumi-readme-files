# Feature 12 — Lecture videos · QA test guide

> For: testers (API, mobile app, admin panel) · technical reference: [feature-12-lecture-videos.md](../features/feature-12-lecture-videos.md) · mobile guide: [feature-12-lecture-videos-mobile.md](feature-12-lecture-videos-mobile.md)

---

## 1. Scope

* Video links per lecture (YouTube, Vimeo, Google Drive, direct link): validate, add, edit, delete, reorder, limit 20.
* Live link validation, provider mismatch hints, duration lookup.
* Videos inside the lecture body (sync).
* Student playback data and the "watched" fallback.
* Admin panel: Lectures → *Videos* row action.

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

| Lecture | Videos |
|---|---|
| Preface | YouTube 08:30 |
| Chapter 1 · Part 1 | YouTube 35:00 |
| Chapter 1 · Part 2 | YouTube 42:10 + «Exercises» 12:40 |
| Production practical | Google Drive (duration unknown) |
| Chapter 2 · Part 1 | Vimeo 25:00 |
| Chapter 2 · Practical | none (`video_missing: true`) |

Postman: `postman/Darsly-Feature-12-Lecture-Videos.postman_collection.json`.

---

## 3. Test cases: link validation

| ID | Provider | URL | Expected (`POST …/videos/validate-url`) |
|---|---|---|---|
| VU-01 | youtube | `https://youtu.be/dQw4w9WgXcQ`, `…watch?v=…&t=30`, `/shorts/…`, `/live/…`, `m.youtube.com` | `valid: true`, `external_id` 11 chars |
| VU-02 | youtube | a Vimeo link | `valid: false`, `VIDEO_URL_PROVIDER_MISMATCH`, `detected_provider: vimeo` |
| VU-03 | youtube | a channel / playlist | `valid: false`, `INVALID_VIDEO_URL` |
| VU-04 | vimeo | `vimeo.com/123`, `vimeo.com/123/abcdef`, `player.vimeo.com/video/123?h=abcdef` | valid; private hash kept |
| VU-05 | google_drive | `/file/d/{id}/view`, `open?id=`, a folder link | valid / valid / invalid with the sharing hint |
| VU-06 | other_link | `https://cdn.x/a.mp4`, `.m3u8`, `http://…mp4`, `https://x/page` | valid, valid, invalid (https), invalid |
| VU-07 | any | `javascript:…`, `ftp://…`, `https://user:pass@…` | invalid |
| VU-08 | — | `provider: rumble` | 422 `VALIDATION_ERROR` |

## 4. Test cases: CRUD & order

| ID | Case | Expected |
|---|---|---|
| VC-01 | Add a YouTube video without duration | 201, `metadata_status: pending` → `resolved` (if the YouTube key is set) or `failed` |
| VC-02 | Add with `duration_seconds` | `manual`; never overwritten by the lookup |
| VC-03 | Add a Vimeo video without duration | duration resolved from Vimeo |
| VC-04 | 21st video | 422 `VIDEO_LIMIT_REACHED` |
| VC-05 | Title omitted | `display_title: الجزء N` |
| VC-06 | Edit the URL | re-validated; duration looked up again unless manual |
| VC-07 | Delete the 2nd of 3 | positions 1, 2 (compacted) |
| VC-08 | Reorder with the full list / a missing id / a foreign id | 200 / 422 / 422 (`video_ids`) |
| VC-09 | Lecture body `videos[]`: one with id, one new, one omitted | updated, added, omitted one deleted; order = list order |
| VC-10 | Lecture body with an unknown video id | 422 `VIDEO_NOT_FOUND` (`videos.N.id`); nothing saved |
| VC-11 | Lecture card | `video_missing` false/true, `counts.videos` |

## 5. Test cases: student playback

| ID | Case | Expected |
|---|---|---|
| VP-01 | Start a view on Chapter 1 · Part 2 | `videos[]` with 2 parts, `players`, `playback.embed_url` |
| VP-02 | Lists / lecture detail | no `playback` data anywhere else |
| VP-03 | Locked / hidden lecture | 403 / 404; no playback data |
| VP-04 | Production practical (unknown duration): heartbeat last part with `video_ended: true` | `completed: true` |
| VP-05 | Heartbeat with a `video_id` of another lecture | 422 `VIDEO_NOT_FOUND` |
| VP-06 | All durations known: watch ≥ 90 % of the total | `completed: true` |

## 6. Test cases: admin panel

| ID | Case | Expected |
|---|---|---|
| AD-01 | Lectures → Videos column | count per lecture |
| AD-02 | Videos row action (repeater) | add / reorder / remove; same validation |

## 7. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Mismatch hint offers switching the source. |
| MB-02 | Each player option of «اختار المشغّل» plays the part. |
| MB-03 | Drive videos play in the webview. |

## 8. Bug report

Lecture / video ids, provider and the exact URL, request and full response (`error_code`, `details`), the test ID and the time.
