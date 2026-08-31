# Architecture

FieldSnap is a React Native / Expo app with a Supabase backend and a Gemini Vision extraction pipeline. 32 routes, ~25,700 lines of TypeScript, 39 runtime dependencies.

The shape of it is set by one fact about the user: they are standing in a plant room, in a basement, on a roof, with one bar of signal and gloves on. Everything below follows from that.

---

## Local-first, sync-second

The app never waits for the network to show a user their own data.

`AppDataContext` is the source of truth. It holds folders, divisions and scans in memory and persists them to device storage under a per-account key (`fieldsnap_appdata_<userId>`, or a separate guest key). Every mutation writes locally and increments a `lastModified` counter.

`SyncContext` watches that counter and pushes to the cloud on a **4-second debounce**, so a burst of edits — renaming three scans in a row — coalesces into one network call instead of three. A `Promise.race` caps any sync at 30 seconds. Sync state is exposed to the UI as `idle | syncing | done | error` with a stage-by-stage progress label, because "uploading photo for Panel DP-B2-01" is a more useful thing to show someone on a bad connection than a spinner.

```
mutation → local write → lastModified++ → [4s debounce] → push
                ↓                                            ↓
          instant UI                              metadata → media → reconcile
```

**Push** upserts folders, then divisions, then scans, uploading photos and video to object storage only where a local URI exists and no remote URL is recorded yet — so a re-sync never re-uploads a file that is already up. It then reconciles deletions by diffing local folder IDs against the cloud set and removing the orphans, with cascade handling the children.

**Restore** is a three-level merge, not a replace. Folders, divisions and scans are matched by ID; conflicts resolve last-write-wins on `updated_at`; anything present only in the cloud is added; anything present only locally survives. Restoring on a second device adds to what is there rather than overwriting it.

---

## The extraction pipeline

Two passes over the same photo, kept deliberately separate and surfaced as two tabs.

| | Raw OCR | Structured extraction |
|---|---|---|
| **Prompt** | "Read every character, preserve line breaks, return only raw text" | "Return only a JSON object with exactly these keys" |
| **Output** | Verbatim text | `Record<fieldKey, string>` |
| **On failure** | Throws — the UI shows a real error | Returns empty strings for every field |
| **Purpose** | The escape hatch | The record |

The split exists because structured extraction will sometimes miss a field the label plainly shows. When it does, the technician switches tabs and reads the raw text rather than re-shooting the plate — and can see exactly what the camera saw, which is also how they tell a bad photo apart from a bad extraction.

The structured pass is prompted with the chosen template's exact field keys and instructed to return an empty string wherever a value is not clearly visible. The response is stripped of markdown fences, parsed, and then **filtered against the requested key set** — any key the model invented is discarded, and any key it omitted is backfilled empty. A malformed response degrades to a blank form the technician can fill by hand, never to a crash and never to a half-populated record with invented values in it.

The two failure modes are treated differently on purpose: OCR throws so the error is visible, because raw text is the diagnostic; structured extraction degrades quietly to empty fields, because a manual form is a working outcome.

A third, smaller prompt names snapshots — a 2–5 word label like "Rating plate" or "Circuit breaker panel" — and fails silently to an empty string, since a missing suggestion is not worth an error message.

---

## Data model

Three levels, because a job site has three levels. A **folder** is a site or a job ("City Hall Renovation"). A **division** is a floor, zone or riser within it. A **scan** is one piece of equipment.

```
users
 └── folders          (name, colour, is_favorite)
      └── divisions   (name, position)
           └── scans  (equipment_type, category, template_data,
                       photo_url, video_url, snapshot_urls,
                       ocr_text, notes, duration)
notifications · push_tokens
```

Six tables. `template_data` is a JSON column keyed by the template's field keys, which is what lets 28 different equipment schemas share one table without 179 columns or a migration every time a template gains a field.

Media lives in two object-storage buckets — `scan-photos` and `scan-videos` — pathed `<userId>/<scanId>/…`, so a user's files are namespaced by their own ID.

---

## Auth

Three ways in, plus a fourth way to not bother.

- **Email OTP** — no password to forget on a job site.
- **Google** via `expo-auth-session`.
- **Apple** via the native button, rendered only where `isAvailableAsync()` says it will work, so it is absent rather than broken elsewhere.
- **Guest** — the full app, locally, no account. Guest data migrates into the account on signup ([see the README](../README.md#three-decisions-worth-defending)).

Session handling carries two deliberate timeouts. The Supabase client is built with a global `fetch` wrapper that aborts after 15 seconds, because the auth library's internal token calls will otherwise hang indefinitely on a flaky mobile network. Session restore at boot races `getSession()` against an 8-second timeout and, on timeout or error, clears storage and lands on the login screen. Both exist because of a real bug: a stale token left the app on a grey screen until the user force-stopped and cleared cache — an unrecoverable state for anyone who is not a developer.

OAuth uses the implicit flow rather than PKCE, which trades a little theoretical rigour for not hitting the code-exchange lock issue in React Native.

---

## Documents and sharing

**PDF export** is a designed document, not a data dump. It renders through the platform print engine from generated HTML: physical page margins set in inches, a `position: fixed` running footer that repeats correctly across page breaks, a branded header with the app icon inlined as a data URI, and per-scan blocks composed of data cards, OCR cards and notes. Images are embedded as data URIs so the file is self-contained the moment it leaves the phone. Custom fields are included alongside built-in ones. A folder exports as one report; a single scan exports on its own.

**Sharing** is a `fieldsnap://` deep link carrying enough context in its query string — name, scan count, equipment types — for the receiving app to render a preview screen before it has fetched anything. The link opens the app if it is installed. It is not a public web viewer, and it is not a permissions model: a recipient without the app gets a link that does not resolve.

---

## Notifications

An in-app notification centre backed by Postgres, with local delivery. `createNotification()` writes a row and immediately schedules a local notification, which means alerts work without server-side push credentials. Device push tokens are registered against the account, so remote push can be added without a client change — but today nothing sends them. Notification writes are non-fatal by design: a failed insert never interrupts the action that triggered it.

---

## Platform work

- **A custom Expo config plugin** strips `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO`, `READ_MEDIA_AUDIO` and `READ_EXTERNAL_STORAGE` from the generated Android manifest to satisfy Google Play's Photo & Video Permissions policy, keeping `READ_MEDIA_VISUAL_USER_SELECTED` for Android 14 partial access.
- **A global font-scale cap** (`maxFontSizeMultiplier: 1`) overrides OS text scaling, replaced by the app's own three-step control.
- **Sentry** with session replay for crash and error reporting on devices that cannot be reached physically.
- **EAS Build** for store artifacts, with environment files excluded from build uploads.

---

## What is not here

- **No automated tests.** Correctness rests on TypeScript, manual verification and crash reporting.
- **No schema migrations in version control.** The database was built through the Supabase console; the table shapes are documented as TypeScript row types in the client, which is a weaker guarantee than a migration history and the first thing worth fixing.
- **No server-side extraction.** The Gemini call is made from the client, which keeps the pipeline simple and means the API key ships with the app — acceptable for a free-tier vision key on a solo product, not acceptable at scale.
- **No remote push sender.**
