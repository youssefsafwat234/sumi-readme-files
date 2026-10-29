# Feature 24 — App & profile settings · Mobile integration guide

> For: mobile developers · API `v1` · technical reference: [feature-24-app-settings.md](../features/feature-24-app-settings.md) · QA guide: [feature-24-app-settings-qa.md](feature-24-app-settings-qa.md) · prerequisites: [Feature 01](feature-01-authentication-mobile.md) (account), [Feature 02](feature-02-teacher-secretary-mobile.md) (secretaries)

The settings page of every role: profile, secretaries (teacher), change instructor (secretary), auto login, light / dark, language, rate the app, logout.

| Item | Student | Teacher | Secretary | Endpoint |
|---|---|---|---|---|
| Profile | ✓ | ✓ | ✓ | `GET /me/profile`, `PUT /me/profile` |
| Password | ✓ | ✓ | ✓ | `PUT /me/password` |
| Secretaries | | ✓ | | `/teacher/secretaries` |
| Change instructor | | | ✓ | `GET /secretary/teachers`, `POST …/{teacher}/use` |
| Auto login / theme / language | ✓ | ✓ | ✓ | `GET` / `PATCH /me/preferences` |
| Rate app | ✓ | ✓ | ✓ | `GET /app/info` |
| Logout | ✓ | ✓ | ✓ | `POST /auth/logout`, `/auth/logout-all` |

---

## 1. Common rules

* Base URL `{{host}}/api/v1`; `Authorization` on everything except `GET /app/info`.
* Envelope `{code, message, errors, data}`; **branch on `error_code`**.
* Language order: the `Accept-Language` header > the saved preference > Arabic.

---

## 2. Preferences

```json
GET /me/preferences
→ { "theme": "system", "locale": "ar", "auto_login": true, "last_used_teacher_role_id": "019f…",
    "app": { "min_supported_version": { "android": "1.2.0", "ios": "1.1.0" },
             "latest_version": { "android": "1.4.2", "ios": "1.4.0" },
             "store_links": { "android": "https://play.google.com/store/apps/details?id=com.darsly.app", "ios": "https://apps.apple.com/app/darsly/id…" },
             "support": { "whatsapp": "+201000000000", "email": "support@darsly.app" } } }
```

`app` is the same object as `GET /app/info` (§6), so the settings screen needs one request. `PATCH /me/preferences` returns it too.

```json
PATCH /me/preferences
{ "theme": "dark", "locale": "en", "auto_login": false }
```

| Field | Values | Notes |
|---|---|---|
| `theme` | `system` · `light` · `dark` | apply locally too |
| `locale` | `ar` · `en` | the confirmation message comes back in the new language |
| `auto_login` | bool | `false`: the **next** sign-in gets a token valid 12 hours, then the app asks to sign in again (the current session isn't cut) |

Preferences are shared by the account's roles.

---

## 3. Profile

`GET /me/profile` → the account (Feature 01 user object: name, email, phone, image, `roles[]` with each **code** for the copy buttons, the teacher's `platform_subscription`) plus, when the secretary role is active:

```json
"secretary": { "linked_teachers": 2, "last_used_teacher": { "role_id": "019f…", "code": "77120458", "name": "د. تسنيم سامي", "is_active": true, "platform_subscription": { "…": "…" } } }
```

Edit: `PUT /me/profile` (Feature 01: `name`, `country_code`, `phone`, `government_id`, `preferred_language`, `image` via `POST` + `_method=PUT`). Email change isn't available.

### Password

```json
PUT /me/password
{ "current_password": "Darsly#User2026", "password": "N3w!Passw0rd#24", "password_confirmation": "N3w!Passw0rd#24" }
```

Other sessions are signed out; this one stays. Google-only accounts → `409 PASSWORD_NOT_SET` (offer "Forgot password" to create one).

---

## 4. Teacher → Secretaries

| Action | Request |
|---|---|
| List | `GET /teacher/secretaries` |
| Link (by the secretary's code) | `POST /teacher/secretaries` (Feature 02) |
| Suspend / reactivate / expiry | `PATCH /teacher/secretaries/{linkId}[/activate|/deactivate]` (Feature 02) |
| Remove | `DELETE /teacher/secretaries/{linkId}` |

Items now include `linked_at` and `secretary {role_id, code, name, email, image_url, last_seen_at}`. Removing a secretary cuts their access to **this** teacher on their next request; they stay signed in for other teachers.

---

## 5. Secretary → Change instructor (L2.3)

```json
GET  /secretary/teachers                       ← linked teachers (state + platform_subscription bar)
POST /secretary/teachers/{teacherRoleId}/use   ← after the L2.3 confirmation
```

`use` remembers the choice (`last_used_teacher_role_id`): on the next launch open that teacher. It never logs out; keep sending the chosen id as `X-Teacher-Id`. A suspended / expired link → `403 SECRETARY_NOT_LINKED_TO_TEACHER`.

---

## 6. Rate app & app info (public)

```json
GET /app/info
→ { "min_supported_version": { "android": "1.2.0", "ios": "1.1.0" },
    "latest_version": { "android": "1.4.2", "ios": "1.4.0" },
    "store_links": { "android": "https://play.google.com/store/apps/details?id=com.darsly.app", "ios": "https://apps.apple.com/app/darsly/id…" },
    "support": { "whatsapp": "+201000000000", "email": "support@darsly.app" } }
```

* Read the values of **your platform** (`android` = Google Play, `ios` = App Store). The admin changes them from the panel at any time: read them on every launch, don't hard-code.
* App version < `min_supported_version.{platform}` → force update; < `latest_version.{platform}` → offer an update.
* «قيّم التطبيق» opens the store link of the platform.

---

## 7. Logout (L2.4)

* `POST /auth/logout` — this device (its push token is removed too).
* `POST /auth/logout-all` — every device.
* Then clear the stored token and call `FirebaseMessaging.instance.deleteToken()`.

---

## 8. Errors to handle

| `error_code` | HTTP | Show |
|---|---|---|
| `VALIDATION_ERROR` | 422 | bad theme / locale, weak password |
| `INVALID_CURRENT_PASSWORD` | 422 | current password wrong |
| `PASSWORD_NOT_SET` | 409 | Google account: use "Forgot password" |
| `SECRETARY_NOT_LINKED_TO_TEACHER` | 403 | the link is suspended / expired |
| `SECRETARY_ROLE_NOT_ACTIVE` | 403 | the secretary role is inactive |

---

## 9. Checklist

- [ ] Theme / language / auto-login saved on the server and applied locally.
- [ ] Codes from `roles[]` with copy buttons (L2.1).
- [ ] Secretary opens on `last_used_teacher_role_id`; switching calls `use`.
- [ ] Force update from `min_supported_version.{android|ios}`; rate button from `store_links`.
- [ ] Logout confirmation (L2.4) then token + Firebase token cleared.
