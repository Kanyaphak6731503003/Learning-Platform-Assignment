## Architecture

### Architecture Overview
The system uses a **lightweight serverless REST API on Vercel in front of Firebase**. It still fits
the $0 budget and one-semester timeline, while adding the HTTP endpoints that Assignment 5 requires
for integration with Team 06 (Submission Module).

The React frontend no longer talks to Firestore directly. Every read and write goes through a small
set of Vercel Serverless Functions, which use the **Firebase Admin SDK** to access Cloud Firestore.
The same API acts as the integration point with Team 06: it proxies their submissions endpoint (with a
fallback when Team 06 is unavailable) and receives their webhooks.

### Architecture Diagram
```text
[ Instructor's React App (Frontend) ]  -- Firebase Hosting
        |
        | HTTPS (JSON), CORS allowlist
        v
[ Team 05 REST API ]  -- Vercel Serverless Functions (repo: learning-platform-api)
   |        |                         |
   |        |                         +--(webhook in)--- [ Team 06 Submission API ]
   |        +--(proxy, 5s timeout,   ---------------->   [ Team 06 Submission API ]
   |            fallback on failure)
   v
[ Cloud Firestore ]  -- accessed only via Firebase Admin SDK (service account)
```

### Components

#### Frontend
* **Technology:** React (Vite) + Tailwind CSS.
* **Role:** Instructor UI: Dashboard, Create/Edit/Detail Assignment, Sections. All data access goes
  through a single service layer (`src/services/api.js`) that calls the Team 05 REST API.
* **Resilience:** Team 06 data is loaded independently of Team 05 data. If Team 06 is down, returns
  404, times out, or sends malformed data, only the submissions panel shows a fallback message.
* **Hosting:** Firebase Hosting.

#### Backend (Team 05 REST API)
* **Technology:** Vercel Serverless Functions (Node.js) + Firebase Admin SDK.
* **Role:** CRUD for assignments and cohorts (sections), the Team 06 submissions proxy, and the
  webhook receiver.
* **Endpoints:**
  * `GET/POST /api/assignments`, `GET/PUT/DELETE /api/assignments/:id`
  * `GET/POST /api/cohorts`, `GET/PUT/DELETE /api/cohorts/:id` (DELETE returns 409 if the cohort
    still has assignments)
  * `GET /api/cohorts/:id/assignments`
  * `GET /api/team06/assignments/:id/submissions` (returns `fallback: true` when Team 06 fails)
  * `POST /api/webhooks/receive` (secret header, idempotent by `submissionId`)
* **Configuration:** Service account and integration secrets are stored as Vercel environment
  variables (`FIREBASE_*`, `TEAM06_BASE_URL`, `TEAM06_INTEGRATION_KEY`, `WEBHOOK_SECRET`,
  `ALLOWED_ORIGINS`).

#### Database
* **Technology:** Cloud Firestore (NoSQL).
* **Role:** Stores Users (Instructors), Cohorts, Assignments, and webhook logs (for idempotency).
* **Access:** Only through the Admin SDK on the API. Firestore Security Rules should deny direct
  client access, since the browser no longer uses the client SDK.

#### Authentication
* **Current state:** Not implemented yet. Records are created for a fixed mock instructor.
* **Planned:** Firebase Authentication (email/password) on the frontend. The frontend sends the
  Firebase ID token in an `Authorization: Bearer` header, and the API verifies it with
  `admin.auth().verifyIdToken()` and scopes every query to that instructor (BR-02).

#### Storage
* **Current state:** Attachments are stored as a URL field (`attachmentUrl`). File upload to Firebase
  Cloud Storage is not implemented yet.

#### External Services
* **Team 06 Submission Module:** consumed through the proxy endpoint above (integration key header)
  and sends `submission.created` webhooks to Team 05.
