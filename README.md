# FacePrint Attend

**A Java-based biometric classroom attendance system for Android.**

This is one Android Studio app with a **Student** section and a **Course Teacher** section, written in **Java + XML only**.

| Layer | Technology | Role |
|-------|------------|------|
| Sign-in | **Firebase Authentication** (email + password, verified email) | identity |
| Shared database | **Cloud Firestore** | source of truth; teacher and students see the same data live |
| Server-side checks | **`firestore.rules`** | roles, duplicates, server-time code expiry, audit trail |
| Local storage | **SQLite** | cache of profile, courses, sessions, sheets and audit logs; offline viewing and CSV export; face template (never uploaded) |
| Face check | CameraX + ML Kit face detection + blink liveness (+ optional TFLite recognition) | front camera |
| Fingerprint | AndroidX BiometricPrompt | phone's built-in sensor |

Everything runs on Firebase's **free Spark plan**. There are no Cloud Functions, no server code and no billing account.

---

## 1. Setup

### 1.1 Firebase project (one time, about 10 minutes)
1. Go to <https://console.firebase.google.com> → **Add project** (Google Analytics is not needed). The plan stays **Spark (free)**.
2. **Add an Android app** with package name `com.example.faceprintattend`. Download **`google-services.json`** and put it in **`app/`**.
3. Go to **Authentication** → Get started → **Email/Password** → Enable.
4. Go to **Firestore Database** → Create database → **production mode** → choose a nearby location (e.g. `asia-south1`, Mumbai). The location cannot be changed later.
5. **Deploy the security rules and the index** from this folder with the Firebase CLI (needs Node.js):
   ```bash
   npm install -g firebase-tools
   firebase login
   firebase use --add            # pick your project
   firebase deploy --only firestore:rules,firestore:indexes
   ```
   *No CLI?* Paste `firestore.rules` into Firestore → Rules → Publish. Then go to Firestore → Indexes → **Single field** → Add exemption: collection ID `attendance`, field `studentUid`, enable **collection group** Ascending. Students' "My attendance" history needs that index.
6. **Make teacher accounts:** in Firestore → **Start collection** `teacherAllowlist` → document ID = the teacher's email in **lower case** (e.g. `rahim@cse.kuet.ac.bd`). Add any field, e.g. `addedAt` = 1. You can add or remove teachers later without rebuilding the app.

### 1.2 App
1. Open the `FacePrintAttend` folder in Android Studio and let Gradle sync. Accept a suggested Android Gradle Plugin upgrade if asked.
2. Optional: set the student email-domain hint `allowed_student_email_domain` in `res/values/config.xml`.
3. Run on a phone with Android 8.0+ (minSdk 26). Use a real phone for the fingerprint and camera checks. An emulator works with a virtual fingerprint (Extended controls → Fingerprint).

**Every account must verify its email** before it can use the app. The rules reject unverified accounts, which stops anyone from registering someone else's address. Teachers register in the app like students. Their email being on `teacherAllowlist` makes them a teacher.

**Optional face recognition:** put a TFLite face-embedding model at `app/src/main/assets/face_model.tflite` (see `assets/README_face_model.txt`). Without it, the face check is detection + blink liveness only. That proves a live person but not *which* person.

---

## 2. Workflow

| # | Who | What happens |
|---|-----|--------------|
| 1 | Teacher | Signs in (verified email, on allowlist) → Teacher section |
| 2 | Teacher | Adds a course → *Start class session* → picks code validity (5–30 min) |
| 3 | Teacher | A 6-character code (with a typo-catching check character) and a QR are shown with a countdown. The live counter updates as students check in |
| 4 | Student | Registers (name + roll; roll is unique app-wide), verifies email, gives consent and enrolls face |
| 5 | Student | *Mark attendance* → types the code or scans the QR → app looks it up in `sessionCodes` |
| 6 | Student | Front-camera face check (one face, close, looking straight, blink) → fingerprint |
| 7 | Student | Record written to Firestore with the **server timestamp**: name, roll, email, course, session/date, time, status, verification results |
| 8 | Rules | Reject a second check-in, an expired/wrong code, another person's roll, a PRESENT without both checks, a client-made time |
| 9 | Teacher | Attendance sheet by course and date (live). Tap a student → mark PRESENT / revoke to ABSENT with a reason → audit log (same transaction, enforced by rules) |
| 10 | Teacher | *End session* → roster students with no record become ABSENT |
| 11 | Teacher | Export **CSV** (Save as… or Share) for one session, or a course summary (P/A per date + %) |

**Exception flow:** a phone without a camera or fingerprint sensor can still submit. The record is **PENDING** until the teacher approves it, and the approval is logged.

Offline: screens open from the SQLite cache and sheets can still be exported. Check-ins and corrections need a connection, because the server checks code expiry with its own clock.

---

## 3. Firestore data model

```
teacherAllowlist/{email}                         console-managed; document exists => teacher
users/{uid}                                      {name, roll, email, createdAt}
rolls/{ROLL}                                     {uid}                (one account per roll)
courses/{courseId}                               {code, name, teacherUid, teacherEmail, createdAt}
courses/{courseId}/roster/{studentKey}           {studentUid|null, name, roll, email, addedBy, addedAt}
sessions/{sessionId}                             {courseId, courseCode, courseName, teacherUid, code,
                                                  sessionDate, startTime, startedAt, expiresAt, endedAt, active}
sessionCodes/{CODE}                              {sessionId, courseId, courseCode, courseName, teacherUid, expiresAt}
sessions/{sessionId}/attendance/{studentKey}     one record per student per session
sessions/{sessionId}/auditLogs/{auditId}         append-only corrections
```
`studentKey` is the student's Firebase UID, or `manual-ROLL` for a student the teacher added by hand.

SQLite cache tables (`database/DbContract.java`): `students` (profiles + roster), `courses`, `sessions`, `attendance`, `audit_logs`, `face_templates` (local only).

### Free-tier budget (Spark)
50,000 document reads, 20,000 writes and 20,000 deletes per day, and 1 GiB storage. A 60-student class uses roughly 200–400 reads and ~70 writes per session, so hundreds of classes per day fit.

---

## 4. Testing

- **Unit tests** (`app/src/test`, 23 tests): code generator, CSV export, sheet building, QR parsing, liveness state machine, face-vector math, status rules. Run them with *Run 'Tests in app'* or `./gradlew test`.
- **Security-rules tests** (`firestore-tests/`, 25 cases): run them against the local emulator (needs Node 18+ and Java 11+):
  ```bash
  cd firestore-tests && npm install && npm test
  ```
- **Manual checklist:** wrong password; unverified email blocked; same student twice → "already checked in"; expired code; printed photo (no blink) fails; fingerprint cancel; no-sensor phone → PENDING → teacher approves → audit log; end session → absentees; CSV opens in Excel; airplane mode → cached sheet still opens and exports.

---

## 5. Limitations

- **Face detection is not recognition.** Without a model the app proves a live face blinked, not whose face it was. The template is enrolled on the student's own phone.
- **BiometricPrompt** proves that *a fingerprint enrolled on that phone* matched, not whose finger it was. The app never sees fingerprint data.
- **Verification flags are reported by the app.** Firestore can't re-run the camera/fingerprint checks, so a modified app could lie about them. The rules still enforce identity, roll ownership, the code, server time, one record per student, and teacher-only corrections. Classroom supervision remains part of the system.
- **Session codes do not prove presence** by themselves: keep them short-lived, rotate the code if it leaks, and watch the live count against the room.
- `expiresAt` is computed on the teacher's phone. A badly wrong teacher clock shortens or lengthens the window; *Stop new submissions* uses server time.

## 6. Versions
Android Gradle Plugin 8.11.1 · Gradle 8.14.3 · compile/target SDK 35 · minSdk 26 · Java 17 · Firebase BoM 33.7.0 (auth, firestore) · CameraX 1.4.1 · ML Kit face-detection 16.1.7 · TFLite 2.16.1 · biometric 1.1.0 · zxing-android-embedded 4.3.0
