# Feature 24 — App & profile settings · QA test guide

> For: testers (API, mobile app) · technical reference: [feature-24-app-settings.md](../features/feature-24-app-settings.md) · mobile guide: [feature-24-app-settings-mobile.md](feature-24-app-settings-mobile.md)

---

## 1. Scope

* Preferences (theme, language, auto login) and the language fallback order.
* Profile with codes and the secretary block; password change.
* Teacher secretaries page (new fields, removal), secretary change instructor.
* Public app info; logout / logout-all (tokens and push tokens).

---

## 2. Setup & data

```bash
php artisan migrate:fresh --seed
```

| Account | Seeded |
|---|---|
| «د. تسنيم سامي» (`tasneem.samy@darsly.test`) | two secretaries: Ahmed Hassan and Omar Khaled |
| Ahmed Hassan | acts for Mona and Tasneem; opens on Mona |
| Omar Khaled | dark theme, Arabic; Google-only (no password) |

Password `Darsly#User2026`. Postman: `postman/Darsly-Feature-24-App-Settings.postman_collection.json`.

---

## 3. Test cases: preferences & language

| ID | Case | Expected |
|---|---|---|
| SP-01 | New account `GET /me/preferences` | `{theme: system, locale: <registration language>, auto_login: true, last_used_teacher_role_id: null, app: <same as /app/info>}` |
| SP-02 | PATCH theme / locale / auto_login | saved; message in the new language |
| SP-03 | `theme: blue` / `locale: fr` | 422 `VALIDATION_ERROR` |
| SP-04 | Omar | `dark`, `ar` |
| SP-05 | Preference `en`, request with `Accept-Language: ar` | Arabic (header wins) |
| SP-06 | Preference `en`, no header | English |
| SP-07 | Public endpoint, no header | Arabic |
| SP-08 | `auto_login: false`, then sign in | token expires after 12 h; the session open before keeps working |

## 4. Test cases: profile & password

| ID | Case | Expected |
|---|---|---|
| PP-01 | Ahmed `GET /me/profile` | 3 role codes; `secretary.linked_teachers: 2`, `last_used_teacher` = Mona |
| PP-02 | A user whose secretary role is inactive | `secretary: null` |
| PP-03 | Change password (right current) | 200; other tokens 401; this one works; login with the new password |
| PP-04 | Wrong current password | 422 `INVALID_CURRENT_PASSWORD` |
| PP-05 | Weak / unconfirmed new password | 422 |
| PP-06 | Omar (Google only) | 409 `PASSWORD_NOT_SET` |

## 5. Test cases: secretaries & change instructor

| ID | Case | Expected |
|---|---|---|
| SC-01 | Tasneem lists secretaries | Ahmed and Omar with `linked_at`, `image_url`, `last_seen_at` |
| SC-02 | Tasneem removes Ahmed | Ahmed's next request with Tasneem's `X-Teacher-Id` → 403; Mona's still works |
| SC-03 | Ahmed `use` Tasneem | 200; `last_used_teacher_role_id` = Tasneem |
| SC-04 | Ahmed `use` Youssef (suspended link) | 403 `SECRETARY_NOT_LINKED_TO_TEACHER` |
| SC-05 | `use` never signs out | the token still works |
| SC-06 | User with an inactive secretary role calls `use` | 403 `SECRETARY_ROLE_NOT_ACTIVE` |

## 6. Test cases: app info & logout

| ID | Case | Expected |
|---|---|---|
| AI-01 | `GET /app/info` without a token | versions per store, store links, support contacts |
| AI-02 | Admin «إعدادات التطبيق»: set Android min 1.2.0, iOS min 1.1.0, latest 1.4.2 / 1.4.0, save | `/app/info` and `/me/preferences → app` return them at once |
| AI-03 | Admin: version `v2`, or latest below the minimum | the field shows an error; nothing saved |
| AI-04 | Support admin opens the page | read-only fields, no Save button |
| AI-05 | Admin clears a field and saves | that value falls back to the env default |
| LO-01 | Logout | token 401; this device's push token removed |
| LO-02 | Logout-all | every token 401; every push token removed; no current device |

## 7. Mobile checks

| ID | Check |
|---|---|
| MB-01 | Dark / light / system switch applies immediately and survives reinstall (from the server). |
| MB-02 | Copy buttons on each role code. |
| MB-03 | L2.3 confirmation, then the app works for the new teacher without re-login. |
| MB-04 | Force-update screen when the app is older than `min_supported_version` of its platform (Android and iOS set apart). |
| MB-05 | L2.4 logout confirmation; no pushes after logout. |

## 8. Bug report

Account, the request and full response (`error_code`, `errors`), headers sent (`Accept-Language`, `X-Teacher-Id`), the test ID and the time.
