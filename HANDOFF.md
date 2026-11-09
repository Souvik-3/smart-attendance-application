# FacePrint Attend: Project Handoff

_Last updated: 2026-10-21 · For the next developer or AI agent picking this project up on another machine._

Read this first, then `README.md` (setup and user-facing docs). This file is the "where things stand and how to continue" view.

---

## 1. What this is

An Android app (one APK, **Java + XML only**) for university classroom attendance with two sections, **Student** and **Course Teacher**. The original specification is `FacePrint_Attend_Final_Project_Summary.md`, which the owner has; it isn't in the repo. The core requirements:

1. Teacher starts a class session → a short-lived **session code** (and QR) is shown.
2. Student signs in, enters the code, passes a **front-camera face check** and a **built-in fingerprint check** (AndroidX BiometricPrompt).
3. Attendance is recorded with name, roll, email, course, session/date, exact time, status and verification results. **No duplicates per session.**
4. The **course teacher** can manually mark PRESENT / revoke to ABSENT with a **mandatory reason**, written to an **audit log** (teacher, time, old → new, reason).
5. Course/date-wise sheets and **one-click CSV export** (Excel-compatible).

**Hard constraints from the owner:**
- Java + XML only (no Kotlin, React or JS backend). Build scripts are Groovy Gradle.
- **Firebase Authentication + Cloud Firestore** as the shared backend, on the **free Spark plan**. That means **no Cloud Functions** and no billing, so all server-side enforcement lives in `firestore.rules`.
- **SQLite** is used for local storage and caching.
- Phone's built-in sensors only (no external fingerprint module).

---

## 2. Current status

| Area | State |
|------|-------|
| App code (46 Java files, ~5700 lines; 15 layouts) | ✅ Feature-complete for the spec |
| Compiles | ✅ Every source file compiled with `javac` against the real `android.jar` (API 34) **plus hand-written stubs for the third-party libraries** (Firebase, CameraX, ML Kit, etc.). ⚠️ **A full Gradle/Android Studio build has never been run** (the original environment had no access to Google's Maven), so expect possible small fixes on first sync. |
| Resources | ✅ Every `@string/@id/@layout…` reference and `R.*` usage resolves; `getString` placeholder counts match |
| Unit tests (`app/src/test`) | ✅ 23/23 pass on the JVM |
| SQLite cache schema | ✅ Executed in SQLite; tables and indexes build |
| `firestore.rules` | ⚠️ Written and reviewed line-by-line against every Firestore call in `AttendanceManager`, but **never executed**: the emulator download was blocked. **First task on a new machine: run `firestore-tests/`** (25 cases). |
| Real device testing | ❌ Not done (camera, BiometricPrompt, Firestore end to end) |
| Firebase project | ❌ Not created. The owner must add `app/google-services.json` (gitignored) |

### Project history (why things look the way they do)
1. **v0, starter zip:** a scaffold using Firebase Realtime Database, with stub camera code.
2. **v1, SQLite-only:** the owner first asked for SQLite as *the* database. Since SQLite can't sync between phones, v1 used signed **QR "attendance receipts"** that the teacher scanned. **That design is gone.** If you see `ReceiptCodec`, `SubmissionDao` or `ReceiptActivity` mentioned anywhere, it's stale.
3. **v2, current:** the owner switched to **Firebase Auth + Firestore** with SQLite as the cache, to stay on the free tier. Students now write directly to Firestore and the teacher sees it live. Session QR now only carries the code (`SessionQr`).

---

## 3. Architecture in one picture

```
 Activities (student/, teacher/, auth/)
        │  callbacks on main thread
        ▼
 attendance/AttendanceManager  ── the ONLY class that talks to Firestore
        │                  │
        │ writes/reads     │ every successful read/listener result is upserted into…
        ▼                  ▼
 Cloud Firestore      database/*Dao  (SQLite cache: DatabaseHelper, DbContract v2)
 (rules enforce           ▲
  everything)             └── screens render from cache first (instant + offline), then live data
```

Key design points:
- **Firestore is the source of truth.** SQLite is a cache. It's wiped on logout (`DatabaseHelper.clearCache()`) except `face_templates`, which never leaves the device.
- **Duplicate prevention:** attendance document ID = `studentKey` inside `sessions/{id}/attendance/`. Students may only *create*, so a second check-in is rejected by the rules, and the app pre-checks inside a transaction for a friendly message.
- **Trusted time:** `markedAt` / `updatedAt` / `changedAt` must equal `request.time` (server timestamp).
- **Code expiry:** `request.time < session.expiresAt`. Codes are unique via create-only `sessionCodes/{CODE}`, and the app retries on collision.
- **Audit trail:** a teacher correction is one transaction that writes `auditLogs/{id}` plus the attendance update with `lastAuditId`. The rules cross-check both with `getAfter()`, so a correction without a matching log is impossible.
- **Roles:** teacher = verified email whose lowercase address is a document in `teacherAllowlist/` (console-managed). Everyone else is a student. The role is cached in SharedPreferences for offline routing (`AuthManager`).
- **Roll uniqueness:** `rolls/{ROLL}` create-only claim, written in the same batch as `users/{uid}`.

Full Firestore layout: see `README.md` §3 and the header of `firestore.rules`.

---

## 4. File map

```
firestore.rules              server-side rules (KEEP IN SYNC with AttendanceManager + FirestorePaths)
firestore.indexes.json       collection-group index on attendance.studentUid (student history)
firebase.json                CLI config (rules, indexes, emulator ports)
firestore-tests/             mocha + @firebase/rules-unit-testing suite (npm test → emulator)
app/build.gradle             deps; google-services plugin applied only if app/google-services.json exists
app/src/main/java/com/example/faceprintattend/
  attendance/AttendanceManager   all Firestore reads/writes + cache updates (754 lines, start here)
  attendance/SheetBuilder        roster + records → full sheet; absentees (pure Java, tested)
  attendance/SessionQr           session QR payload "FPS1;CODE" (pure, tested)
  attendance/AttendanceStatus    status/source/face-method string constants (shared with rules!)
  firebase/FirestorePaths        collection names
  firebase/FirestoreMapper       DocumentSnapshot → models
  auth/  AuthManager (routing, roles), Login, Register, VerifyEmail
  biometric/  FaceVerificationActivity (CameraX+ML Kit), LivenessTracker (blink state machine),
              FaceRecognizer (optional TFLite), FaceMath, FaceDetectionManager, BiometricAuthManager
  database/  DbContract (schema v2), DatabaseHelper, Student/Course/Session/Attendance/AuditLog/FaceTemplate DAOs
  models/    Student (studentKey = uid or "manual-ROLL"), Course, ClassSession, AttendanceRecord, AuditLogEntry, VerificationResult
  student/   StudentDashboard, Profile (consent + face enrollment), SessionCode (check-in flow)
  teacher/   TeacherDashboard, CreateSession (live session screen), AttendanceSheet (live + export),
             ManualAttendance (override), AuditLog, TeacherGuard (biometric/password re-confirm)
  ui/ RowAdapter (shared list), UiUtils     utils/ CsvExportUtils, FileExportHelper (SAF + share), CodeGenerator, DateTimeUtils, QrUtils
app/src/main/res/values/config.xml   durations, face thresholds, email-domain hint
app/src/main/assets/README_face_model.txt   how to add face_model.tflite (optional recognition)
```

---

## 5. Getting running on a new machine

1. Install **Android Studio** (recent), JDK 17 (bundled), **Node 18+**, **Java 11+** (for the emulator).
2. Unzip, then open `FacePrintAttend/` in Android Studio → Gradle sync. If the AGP version (8.11.1) is not supported by your Studio, let the Upgrade Assistant change it. Then fix any compile errors that the real libraries reveal; see §2 on why this is possible.
3. **Run the rules tests before anything else:**
   ```bash
   cd firestore-tests && npm install && npm test
   ```
   Fix `firestore.rules` until all 25 pass. If you change rules, update the tests.
4. Follow `README.md` §1 to create the Firebase project, add `google-services.json`, deploy rules and indexes, and add a `teacherAllowlist` doc.
5. Unit tests: `./gradlew test` (or right-click `app/src/test` → Run).
6. Device test with **two phones** (teacher + student) using the manual checklist in `README.md` §4.

---

## 6. Rules you must not break

- **Field names and constants are a contract between three places:** `AttendanceManager` (writes), `FirestoreMapper` (reads) and `firestore.rules` (validation). Rename in all three plus `firestore-tests/rules.test.js`.
- Student check-in doc must contain: `studentUid, name, roll, email, courseId, courseCode, courseName, sessionId, sessionCode, sessionDate, status, markedAt(serverTimestamp), updatedAt(serverTimestamp), faceDetected, livenessPassed, faceCheckPassed, faceMethod, faceScore, fingerprintPassed, source='STUDENT_APP', markedBy, manualOverride=false` and **no** `lastAuditId`.
- Never add a Firestore call outside `AttendanceManager`. Screens should stay cache-first.
- Spark plan: no Cloud Functions or scheduled jobs. Watch read counts (listeners on large collections, roster per session). Free daily limits: 50k reads / 20k writes.
- Face templates stay on the device (privacy/consent). Don't upload them.

---

## 7. Known issues / TODO (priority order)

1. **[High] Verify on real tooling:** Gradle build, rules emulator tests, two-device end-to-end test. Nothing has been executed on Android yet.
2. **[Medium] Verification flags are reported by the app.** The rules can't re-run face or fingerprint checks, so a tampered app could submit `faceCheckPassed=true`. Options: Firebase **App Check** with Play Integrity (works on Spark) and enforcing `request.app` in rules is the best free mitigation. Add it before real deployment.
3. **[Medium] `expiresAt` uses the teacher's phone clock** (`AttendanceManager.createSessionAttempt` / `regenerateAttempt`). A better design is to store `durationMinutes` and enforce `request.time < startedAt + duration.value(d,'m')` in the rules (server-only time). *Stop submissions* already uses server time.
4. **[Low] Manual-roster vs. registered student:** a teacher-added `manual-ROLL` student who later registers gets a second key (their UID). `SheetBuilder` now merges them by roll in the sheet and the absentee list, but two attendance docs could exist if the teacher manually marked the `manual-` key *and* the student checked in. Possible fix: on check-in, have the rules or teacher UI detect a same-roll `manual-` record.
5. **[Low] `endSession` batch cap** is 450 absentees (Firestore's 500-write batch limit). Fine for classes; chunk it if needed.
6. **[Low] Liveness is blink-only.** A video replay could pass. Optional: random head-turn challenge (ML Kit gives `headEulerAngleY`) in `LivenessTracker`.
7. **[Low] No instrumentation/UI tests** (Espresso) yet. Add them for the check-in and correction flows.
8. **[Nice] Face recognition model:** none bundled (licensing). Pick a MobileFaceNet TFLite, tune `face_match_threshold_percent`.

---

## 8. Decisions log

| Decision | Why |
|----------|-----|
| Cloud Firestore (not Realtime DB) | Richer rules (`getAfter`, `let`, transactions), collection-group queries, server timestamps; free tier is enough |
| Subcollections `sessions/{id}/attendance` + `auditLogs` | Natural per-session security checks and listeners |
| Rules instead of Cloud Functions | Spark plan has no Functions |
| Email verification required for all | Stops registering someone else's (e.g. a teacher's) email |
| Teacher role via `teacherAllowlist` collection | Changing teachers needs no app rebuild; can't be self-assigned |
| SQLite cache cleared on logout (except face templates) | Shared/classroom phones shouldn't leak another user's data |
| PENDING status for missing hardware | Spec §8 "teacher-approved exception flow" |
| CSV with BOM, CRLF, all-quoted, formula-injection guard | Opens cleanly in Excel; safe |

---

## 9. Prompt to give a new AI agent

> You are continuing the FacePrint Attend Android project (Java/XML, Firebase Auth + Firestore on the free Spark plan, SQLite cache). Read `HANDOFF.md` and `README.md` first. Do the HANDOFF §5 steps in order: Gradle sync and fix compile issues; run `firestore-tests` against the emulator and make all 25 pass; then work through §7 from the top. Keep `firestore.rules`, `AttendanceManager`, `FirestoreMapper` and `rules.test.js` in sync for every data change. Don't introduce Kotlin, Cloud Functions or paid Firebase features.
