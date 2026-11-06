# Product Requirements Document

## 1. Product Overview

### Product Name
**Learning Platform-Assignment**

### Problem Statement
In educational settings without a centralized system, assignment details get lost across scattered communication channels (email, syllabi, chat apps like WhatsApp or Discord). The lack of a single source of truth leads to missed deadlines, misunderstood requirements, and high administrative overhead for instructors who must repeatedly answer the same questions. This product gives instructors one consistent, organized place to record every assignment's due date and criteria, so they can manage their courses reliably and stop re-explaining the same details across scattered channels.

### Target Users
**Instructors** - the sole Target User of this system. Instructors are the only people who create accounts, log in, and use the application in any way.

### Product Goal
Give instructors a single, reliable place to create, organize, and track assignments with clear due dates and criteria, reducing missed deadlines and repetitive student questions delivered as a lightweight MVP buildable by a small student developer team in one semester.

---

## 2. Scope

### In Scope (MVP)
- Instructor account creation and login.
- Instructor-owned Cohorts (classes/sections) to organize assignments.
- Instructor assignment creation, editing, and deletion with due dates, criteria, and an optional attachment.
- 3 core screens covering the instructor's main journey.
- **All core CRUD operations go through the Team 05 REST API (Vercel Serverless Functions + Firebase Admin SDK).**
- Integration with Team 06 (Submission Module): a submissions proxy with a fallback when Team 06 is unavailable, and a webhook receiver.
- At least 7 automated tests covering essential paths and validation.

### Out of Scope (MVP)
- Student accounts, login, or self-enrollment.
- Any public or unauthenticated access to assignment data (no share links, no read-only views).
- Submission workflow (students uploading/submitting work) - moved to Future Improvements.
- AI-assisted features - moved to Future Improvements.
- Native mobile apps.
- Push/email notification systems.

---

## 3. User Roles

### Instructor (the only role in MVP)
- Creates and manages Cohorts.
- Creates, views, edits, and deletes Assignments with due dates and criteria.
- Is the only person who can access any data in the system everything is scoped to the authenticated Instructor who owns it.

---

## 4. User Journey

### Main Journey (Instructor)
1. **Sign Up/Log In** - Instructor authenticates via email/password.
2. **Create/Edit Assignment** - Instructor sets title, description, due date, and criteria for an Assignment, and it is saved immediately as the authoritative record.

---

## 5. Functional Requirements

### FR-01 Authentication
Instructors can sign up and log in with email/password via Firebase Auth.

### FR-02 Cohort Management
Instructors can create, rename, and delete a Cohort (a class/section) to organize their Assignments.

### FR-03 Assignment Creation
Instructors can create an Assignment within a Cohort, including title, description, due date, criteria/rubric text, and an optional file attachment (e.g., a rubric or instructions document).

### FR-04 Assignment Editing & Deletion
Instructors can update or delete Assignments they own.

---

## 6. Non-Functional Requirements

### NFR-01 Performance
Core screens (Assignment view) should load in under 2 seconds on a typical broadband connection, given expected low concurrent usage (single-classroom scale).

### NFR-02 Security / Data Isolation
- One Instructor's Cohorts and Assignments must never be readable or editable by another Instructor.
- No data in the system is accessible without authentication. There is no unauthenticated read or write path of any kind.
- Enforced via Firestore Security Rules (see Section 12).

### NFR-03 Availability
Best-effort availability appropriate for a free-tier academic project, no formal SLA. Firebase Hosting/Firestore's standard uptime is acceptable.

### NFR-04 Cost
The system must run entirely within free-tier limits (Firebase Spark plan) for the duration of the semester see CON-01.

---

## 7. Business Rules

### BR-01 Account Creation
Public sign-up creates an Instructor account. There is no self-service Student account type in MVP.

### BR-02 Ownership & Isolation
An Instructor can only create, view, edit, or delete their own Cohorts and Assignments. Cross-instructor access is not permitted anywhere in the app.

### BR-03 No External Access
The system has no mechanism for any unauthenticated party to view, create, or modify data. Every read and write requires a valid Instructor session; there are no public links, tokens, or externally shared views in MVP.

---

## 8. Data Model

### User (Instructor)
Stores authentication and profile data for instructors the only account type created in MVP.
- **Fields:** ID (PK = Firebase Auth UID), FullName, Email (Unique)
- **Relationships:** 1-to-Many with Cohort
- **Permissions:** Create: Public (Sign up). Read/Update/Delete: Self only.

### Cohort
Represents a class/section owned by one Instructor.
- **Fields:** ID (PK), Name, InstructorID (FK → User)
- **Relationships:** Many-to-1 with User (Instructor); 1-to-Many with Assignment
- **Permissions:** Create/Read/Update/Delete: Owning Instructor only. No other party can access a Cohort's data.

### Assignment
An assignment created by an Instructor within a Cohort.
- **Fields:** ID (PK), CohortID (FK → Cohort), Title, Description, Criteria, DueDate, AttachmentURL (optional), CreatedAt
- **Relationships:** Many-to-1 with Cohort
- **Permissions:** Create/Read/Update/Delete: Owning Instructor only. No other party can access an Assignment's data.

---

## 9. Architecture

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

---

## 10. Technology Stack

| Layer | Technology | Reason |
| --- | --- | --- |
| **Frontend** | React | Team-specified; fast to build the 3 core instructor screens |
| **Backend** | Vercel Serverless Functions + Firebase Admin SDK | Provides the REST endpoints required for inter-team integration (Assignment 5) while staying on free tiers |
| **Database** | Firestore | Team-specified NoSQL store; a good fit since the MVP data model is simple (User → Cohort → Assignment) |
| **Auth** | Firebase Auth | Team-specified; fast, free-tier email/password auth for Instructors |
| **Hosting** | Firebase Hosting (frontend), Vercel (API) | Free static hosting for the React app; Vercel deploys the API automatically on every push |

---

## 11. Data Access / API Endpoints

The React frontend performs every operation through the Team 05 REST API, which reads and writes Cloud Firestore with the Firebase Admin SDK. The DA numbering is unchanged; each operation now maps to an HTTP endpoint.

* **DA-01 - Create Cohort:** `POST /api/cohorts`
* **DA-02 - List Cohorts:** `GET /api/cohorts`
* **DA-03 - Rename Cohort:** `PUT /api/cohorts/:cohortId`
* **DA-04 - Delete Cohort:** `DELETE /api/cohorts/:cohortId`. Returns `409 Conflict` while any assignment still belongs to the cohort, so no orphaned assignments remain.
* **DA-05 - Create Assignment:** `POST /api/assignments`
* **DA-06 - List Assignments:** `GET /api/assignments`, or `GET /api/cohorts/:cohortId/assignments` for one cohort
* **DA-07 - Fetch Assignment:** `GET /api/assignments/:assignmentId`
* **DA-08 - Update Assignment:** `PUT /api/assignments/:assignmentId`
* **DA-09 - Delete Assignment:** `DELETE /api/assignments/:assignmentId`

Inter-team endpoints (Assignment 5):

* **INT-01 - Fetch Team 06 submissions:** `GET /api/team06/assignments/:assignmentId/submissions`. Proxies Team 06 with a 5-second timeout. On any failure it returns `200` with `fallback: true` and an empty list, so the frontend can degrade gracefully.
* **INT-02 - Receive webhook:** `POST /api/webhooks/receive`. Requires the `x-webhook-secret` header; idempotent by `submissionId`.

---

## 12. Security

### Authentication

* **Current state:** Not implemented yet. The frontend creates records for a fixed mock instructor.
* **Planned:** Firebase Auth (email/password) on the frontend. The frontend sends the Firebase ID token in an `Authorization: Bearer` header, and the API verifies it with `admin.auth().verifyIdToken()`.

### Authorization

* **Planned:** The API scopes every query and write to the verified instructor's UID (BR-02). Until then, the REST API does not enforce ownership.
* CORS restricts browser access to the Team 05 frontend origins (`api/_cors.js`, `ALLOWED_ORIGINS`).
* Firestore is accessed only through the Admin SDK on the server; Firestore Security Rules should deny direct client access.
* The webhook endpoint is protected by a shared secret header.

### Data Protection

* No password or password hash is ever stored in Firestore or exposed via any read path. Firebase Auth owns that data entirely.
* No database query returns another Instructor's data.
* Uploaded attachments are size-capped at 5MB and access-controlled via Storage security rules tied to the owning Instructor.

---

## 13. Error Handling

### Expected Errors (HTTP)

* Document doesn't exist → `404 Not Found`
* Deleting a cohort that still has assignments → `409 Conflict`
* Webhook with a missing or wrong secret → `401 Unauthorized`
* Team 06 down, `404`, or timeout → `200` with `fallback: true` (frontend shows a fallback message; Team 05 data is unaffected)

### Failure Scenarios

* **File upload failure** (size/type) → reject client-side and server-side (Storage rules) with a clear error message.

---

## 14. Deployment

### Development

* Firebase Emulator Suite (Auth, Firestore, Storage) for local development and testing, avoiding any cost during dev.
* Feature branches with PR review before merging to `main`.

### Production

* Firebase Hosting for the React frontend (`firebase deploy`).
* **Firestore, Auth, and Storage rules deployed seamlessly via Firebase CLI.**
* Team 05 API deployed on Vercel automatically on every push to `main` of `learning-platform-api`.
* **The entire production environment runs on free tiers (Firebase Spark plan and Vercel Hobby plan).**

---

## 15. Constraints

* **Budget:** $0 deployment budget (CON-01).
* **Time:** One semester.
* **Team:** 3-5 student developers.
* **Free Tier:** All services must operate strictly within the **Firebase Spark Plan** free-tier quotas.

---

## 16. Risks

| Risk | Impact | Mitigation |
| --- | --- | --- |
| **REST API is not yet authenticated** | **High** - Until FR-01 token verification is added, anyone who knows the API URL can read or change data. | Add Firebase ID token verification to every endpoint before real use; CORS already limits browser access to known origins. |
| **Small team / one semester timeline** | **Medium** - Scope creep could prevent MVP completion. | Strict adherence to Section 2 scope; defer anything not listed to Section 18. |

---

## 17. Acceptance Criteria

### MVP is complete when:

* [ ] Instructors can sign up, log in, create a Cohort, and create an Assignment with a due date and criteria.
* [ ] The 3 core screens (Login, Home, Create/Edit Assignment) are implemented and functional.
* [ ] Instructors can view, edit, and delete their own Cohorts and Assignments from the Home.
* **[ ] The React frontend executes all data operations through the Team 05 REST API and keeps working when Team 06 is unavailable.**
* [ ] At least 7 automated tests cover essential paths (auth, Cohort/Assignment CRUD, and security-rule enforcement for cross-instructor isolation).
* [ ] Firestore/Storage Security Rules prevent one Instructor from reading/writing another Instructor's data (verified by tests).
* **[ ] The entire system runs on free tiers (Firebase Spark and Vercel Hobby) with a $0 deployment cost.**

---

## 18. Future Improvements

* **Read-only, link-based sharing of a Cohort's Assignments** (no login required) so students can view due dates and criteria directly - deferred out of MVP to keep the system purely instructor-facing at this stage.
* **Student accounts, self-enrollment (via join code), and a full submission workflow** (file upload, tracking, instructor review) - this was part of the original data model concept but is deferred out of MVP now that Instructors are the sole target user.
* Notification system (email/push) for upcoming due dates.
* Instructor gradebook/analytics Home.
* Native mobile app.
* Plagiarism detection integration.
* In-app messaging between instructors and students (replacing WhatsApp/Discord use).
* Bulk CSV import for Cohort rosters.