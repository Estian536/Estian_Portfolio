# Inspecoza

**Inspecoza** is a native Android inspection/audit app built by **Ayesa Core Services**. It lets field
staff fill out dynamic, admin-defined inspection forms — with photos, GPS location, and signatures —
entirely **offline**, and syncs everything to a Google Sheets / Google Apps Script backend as soon as a
connection is available. Every completed inspection can also be exported and shared as a formatted PDF
report.

Under the hood, Inspecoza is a **hybrid app**: a thin native Android shell (Java) hosts a full-screen
`WebView` that runs the actual UI (HTML/CSS/JS). The native shell exposes device features — camera,
GPS, encrypted local storage, background sync — to the web layer through a JavaScript bridge.

---

## 1. How the app is put together

```
Android (Java)                          WebView (HTML/CSS/JS)
────────────────────                    ──────────────────────
MainActivity                    ⇄        index.html / index.js      (Home / Dashboard)
 ├─ AndroidBridge  (JS bridge)  ⇄        AreaPage/                  (Area → form list / submissions)
 ├─ LocationHelper                       DynamicForm/               (Renders & fills a form)
 ├─ SyncManager                          DynamicView/               (Read-only submission viewer)
 ├─ FormManager
 ├─ StorageHelper
 ├─ CryptoManager
 ├─ PdfBuilder
 └─ SubmissionSyncWorker (WorkManager)
```

- **`MainActivity.java`** — app entry point. Boots the `WebView`, wires up all helper classes,
  handles the camera/file chooser flow, permissions, back-button handling (delegated to JS first),
  and loads the encrypted `config.json`.
- **`AndroidBridge.java`** — the `window.AndroidInterface` object injected into the WebView. Every
  native capability the web UI needs (save a submission, take a photo, get GPS, sync, export PDF,
  read/write forms, etc.) is exposed here as a `@JavascriptInterface` method.
- **`FormManager.java`** — downloads form **templates** from the Google Apps Script backend, stores
  them encrypted in a `tempForms/` staging folder, then merges them into the live `forms/` folder
  (adding new templates, updating changed ones, deleting retired ones).
- **`SyncManager.java`** — uploads completed submissions to the backend, tracks pending/failed/sent
  counts, and prevents duplicate sends via a confirmed-ID registry.
- **`SubmissionSyncWorker.java`** — a `WorkManager` background job that guarantees pending
  submissions still get uploaded even if the user closes the app before the foreground sync finishes
  (runs only when the device has network connectivity).
- **`StorageHelper.java`** — sets up the app's folder structure and cleans up old/temporary files.
- **`CryptoManager.java`** — AES‑GCM encryption backed by the hardware **Android Keystore**. Config,
  form templates, submissions, and drafts are all encrypted at rest on the device.
- **`LocationHelper.java`** — requests permission and captures a GPS/network location fix for
  location-tagging a submission.
- **`PdfBuilder.java`** — renders a completed submission into a printable PDF report (via an
  off-screen `WebView`) and hands it to the Android share sheet.

## 2. The web UI (what the user actually sees)

| Screen | File | Purpose |
|---|---|---|
| **Home / Dashboard** | `assets/index.html` + `index.js` | Splash/loading screen, side menu, "Activate App" flow, form-template updater, and a live submission-sync dashboard (Total / Pending / Sent / Failed counts, manual sync, cleanup of old sent submissions). |
| **Area Page** | `assets/AreaPage/` | Opened from the menu for a given inspection "area". Has two tabs: **Add** (list of forms available to fill in that area) and **View** (past submissions for that area, with per-form submission counts). |
| **Dynamic Form** | `assets/DynamicForm/` | Renders a form from its JSON template (admin-defined fields, sections, photo capture, geolocation capture, signature, etc.), validates input, autosaves a draft, and saves the finished submission. |
| **Dynamic View** | `assets/DynamicView/` | Read-only viewer for a previously submitted inspection — used to review a submission or trigger the "Export & Share PDF" action. |

Forms are **not hard-coded** — they are JSON templates pulled from the backend, so new inspection
forms/checklists can be rolled out to every device without an app update.

## 3. End-to-end user flow

1. **First launch / Activation** — the user (or an admin) enters a **product key** in the "Activate
   App" modal. This calls a Google Apps Script endpoint (its URL is Base64-obfuscated in the app and
   resolved at runtime) to activate the device and receive its server configuration.
2. **Config & form sync** — once activated, `config.json` (server URLs + API token) is saved
   encrypted on-device. The app then downloads the current set of form templates and merges them
   into local storage.
3. **Device naming** — on first run the user is prompted to name the device (used to tag every
   submission with `deviceId` / `deviceName`).
4. **Permissions** — camera (for photo attachments) and location (for GPS-tagging inspections) are
   requested as needed; a battery-optimization exemption is also requested so background sync isn't
   killed by the OS.
5. **Doing an inspection**:
   - From the **Home** screen the user opens the side menu and picks an **Area**.
   - In the **Area Page**, the user taps a form under the **Add** tab.
   - The **Dynamic Form** screen renders that form's fields. The user fills it in, attaches
     photo(s) via the camera, and the form can capture the current GPS location. Progress is
     autosaved as a **draft** so nothing is lost if the app is closed mid-form.
   - On submit, the completed record is written to local storage as an **encrypted JSON
     submission** and queued for upload.
6. **Syncing**:
   - If the device is online, the submission uploads immediately.
   - A `WorkManager` background job also runs, so anything still pending gets delivered once
     connectivity returns — even if the app isn't open.
   - The **Home dashboard** shows live Total / Pending / Sent / Failed counters and a manual
     **"Sync Now"** button.
7. **Reviewing / exporting**:
   - Under the **View** tab of an Area Page, the user can see past submissions for each form.
   - Opening one loads the **Dynamic View** screen, from which the user can **export and share a
     PDF** report of that inspection via the Android share sheet.
8. **Housekeeping**:
   - The **"Sent Submissions"** stat on the dashboard opens a **Cleanup** modal, letting the user
     delete already-synced submissions older than a chosen interval to free up device storage
     (pending/failed submissions are never touched).
   - Stale drafts, temp camera images, and cached shared PDFs are cleaned automatically at startup.

## 4. Data & security model

- All persistent app data — **config, form templates, drafts, and submissions** — is encrypted at
  rest with **AES‑GCM**, using a key generated inside the Android Keystore (the raw key material
  never leaves secure hardware). Legacy unencrypted files are transparently migrated to encrypted
  storage the first time they're read.
- On-device folders (under app-external storage): `config/`, `forms/`, `tempForms/`, `images/`,
  `submissions/`, `tempSubmissions/`.
- Submissions are filenamed `SUB_<TemplateName>_<DeviceId>_<timestamp>.json` and tagged with
  `deviceId`, `deviceName`, `timestamp`, `syncStatus` (`pending` / `sent` / `failed`), and optional
  `mail_to` / `mail_cc` fields pulled from the form data (used server-side for email delivery of
  reports).
- The backend is a **Google Apps Script web app** (acting as a lightweight API in front of a Google
  Sheet/Drive): one endpoint serves form templates (`?action=getAll`), another accepts submission
  uploads. All requests are authenticated with a per-install API token stored in the encrypted
  config.

## 5. Tech stack

- **Platform:** Native Android (Java), `minSdk`/`targetSdk` per `app/build.gradle.kts`
- **UI layer:** Single `WebView` hosting a vanilla HTML/CSS/JS front end (no external JS framework)
- **Native ↔ Web bridge:** `AndroidBridge` via `addJavascriptInterface`
- **Background work:** `androidx.work.WorkManager` (`SubmissionSyncWorker`)
- **Storage:** App-external files dir, AES‑GCM encrypted via Android Keystore
- **Backend:** Google Apps Script (Sheets/Drive-backed), consumed over HTTPS with a bearer-style
  API token
- **PDF generation:** Off-screen `WebView` render → PDF, shared via `FileProvider`

## 6. Project layout

```
Inspecoza/
├─ app/
│  ├─ src/main/
│  │  ├─ java/com/ayesacoreservices/inspecoza/   # Native Android layer (see table above)
│  │  ├─ assets/
│  │  │  ├─ index.html / index.js / index.css    # Home / dashboard
│  │  │  ├─ AreaPage/                            # Per-area form list + submission history
│  │  │  ├─ DynamicForm/                         # Form renderer / data entry
│  │  │  ├─ DynamicView/                         # Read-only submission viewer
│  │  │  └─ Media/                                # Loading animation, etc.
│  │  ├─ res/                                     # Android resources (theme, icons, XML config)
│  │  └─ AndroidManifest.xml
│  └─ build.gradle.kts
├─ gradle/, settings.gradle.kts, build.gradle.kts  # Gradle build config
└─ README.md
```

## 7. Permissions used

| Permission | Why |
|---|---|
| `CAMERA` | Attach inspection photos |
| `ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION` | GPS-tag a submission |
| `INTERNET` / `ACCESS_NETWORK_STATE` | Sync forms & submissions with the backend |
| `READ_EXTERNAL_STORAGE` (≤ Android 12) | Legacy file access for older OS versions |

---