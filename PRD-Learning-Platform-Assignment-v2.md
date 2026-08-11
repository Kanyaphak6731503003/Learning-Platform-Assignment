# Product Requirements Document

## 1. Product Overview

### Product Name
**Learning Platform-Assignment**

### Problem Statement
In educational settings without a centralized system, assignment details get lost across scattered communication channels (email, syllabi, chat apps like WhatsApp or Discord). The lack of a single source of truth leads to missed deadlines, misunderstood requirements, and high administrative overhead for instructors who must repeatedly answer the same questions. This product gives instructors one consistent, organized place to record every assignment's due date and criteria, so they can manage their courses reliably and stop re-explaining the same details across scattered channels.

### Target Users
**Instructors** — the sole Target User of this system. Instructors are the only people who create accounts, log in, and use the application in any way.

### Product Goal
Give instructors a single, reliable place to create, organize, and track assignments with clear due dates and criteria, reducing missed deadlines and repetitive student questions — delivered as a lightweight MVP buildable by a small student developer team in one semester.

---

## 2. Scope

### In Scope (MVP)
- Instructor account creation and login.
- Instructor-owned Cohorts (classes/sections) to organize assignments.
- Instructor assignment creation, editing, and deletion with due dates, criteria, and an optional attachment.
- 3 core screens covering the instructor's main journey.
- REST API with at least 6 operations, documented as a shared contract.
- At least 7 automated tests covering essential paths and validation.

### Out of Scope (MVP)
- Student accounts, login, or self-enrollment.
- Any public or unauthenticated access to assignment data (no share links, no read-only views).
- Submission workflow (students uploading/submitting work) — moved to Future Improvements.
- Inter-team webhook integrations and AI-assisted features — moved to Future Improvements.
- Native mobile apps.
- Push/email notification systems.

---

## 3. User Roles

### Instructor (the only role in MVP)
- Creates and manages Cohorts.
- Creates, views, edits, and deletes Assignments with due dates and criteria.
- Is the only person who can access any data in the system — everything is scoped to the authenticated Instructor who owns it.

---

## 4. User Journey

### Main Journey (Instructor)
1. **Sign Up / Log In** — Instructor authenticates via email/password.
2. **Create/Edit Assignment** — Instructor sets title, description, due date, and criteria for an Assignment, and it is saved immediately as the authoritative record.

---

## 5. Functional Requirements

### FR-01 — Authentication
Instructors can sign up and log in with email/password via Firebase Auth.

### FR-02 — Cohort Management
Instructors can create, rename, and delete a Cohort (a class/section) to organize their Assignments.

### FR-03 — Assignment Creation
Instructors can create an Assignment within a Cohort, including title, description, due date, criteria/rubric text, and an optional file attachment (e.g., a rubric or instructions document).

### FR-04 — Assignment Editing & Deletion
Instructors can update or delete Assignments they own.

---

## 6. Non-Functional Requirements

### NFR-01 Performance
Core screens (Assignment view) should load in under 2 seconds on a typical broadband connection, given expected low concurrent usage (single-classroom scale).

### NFR-02 Security / Data Isolation
- One Instructor's Cohorts and Assignments must never be readable or editable by another Instructor.
- No data in the system is accessible without authentication — there is no unauthenticated read or write path of any kind.
- Enforced via Firestore Security Rules (see Section 12).

### NFR-03 Availability
Best-effort availability appropriate for a free-tier academic project; no formal SLA. Firebase Hosting/Firestore's standard uptime is acceptable.

### NFR-04 Cost
The system must run entirely within free-tier limits (Firebase Spark plan) for the duration of the semester — see CON-01 and the related risk in Section 16 regarding Cloud Functions.

---

## 7. Business Rules

### BR-01 — Account Creation
Public sign-up creates an Instructor account. There is no self-service Student account type in MVP.

### BR-02 — Ownership & Isolation
An Instructor can only create, view, edit, or delete their own Cohorts and Assignments. Cross-instructor access is not permitted anywhere in the app.

### BR-03 — No External Access
The system has no mechanism for any unauthenticated party to view, create, or modify data. Every read and write requires a valid Instructor session; there are no public links, tokens, or externally shared views in MVP.

---

## 8. Data Model

### User (Instructor)
Stores authentication and profile data for instructors — the only account type created in MVP.
- **Fields:** ID (PK, = Firebase Auth UID), FullName, Email (Unique)
- **Relationships:** 1-to-Many with Cohort
- **Permissions:** Create: Public (Sign up). Read/Update/Delete: Self only.

*(The earlier draft's `Role` enum (Instructor/Student) and the `Enrollment`/`Submission` entities remain removed from the MVP data model, since students are not application users in this scope — see Section 18 for that functionality as a future phase.)*

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
A Firebase-based architecture matching the team's constraints (3–5 student developers, one semester, zero deployment budget). All Cohort and Assignment data — every create, read, update, and delete — goes through a single path: the authenticated REST API (Firebase Cloud Functions). The React frontend never writes or reads Firestore directly. This keeps authorization logic in one place (the Cloud Functions layer) instead of split between client-side Firestore calls and server-side REST calls, which avoids the risk of the two paths enforcing different rules. The frontend does talk directly to Firebase Auth (to sign in and obtain an ID token) and directly to Firebase Storage (only for uploading attachment files, which is a separate concern from Cohort/Assignment data — see the Storage component below).

### Architecture Diagram

[Instructor's React App] --(Firebase SDK)--> [Firebase Auth]  (sign in, get ID token)
       |
       +--(Firebase SDK, direct upload)-----> [Firebase Storage]  (attachment files only)
       |
       +--(HTTPS REST calls, authenticated)--> [Firebase Cloud Functions] --> [Firestore DB]
                                                  (all Cohort/Assignment CRUD)

### Components

#### Frontend
React (Create React App or Vite + React), used only by Instructors.

#### Backend
A single layer: Firebase Cloud Functions expose the authenticated REST API (see Section 11), which is the only path for all Cohort and Assignment CRUD. The frontend does not call Firestore directly for this data — this keeps ownership/authorization checks in one codebase instead of duplicating them across a client SDK path and a server path.

#### Database
Firestore (NoSQL document database), with collections mirroring the entities in Section 8 (users, cohorts, assignments). The simplified, mostly two-level structure (Cohort → Assignment) fits Firestore's document model well. Firestore is only ever accessed by the Cloud Functions layer (via the Firebase Admin SDK) — the React frontend has no direct read/write path to it.

#### Authentication
Firebase Auth (email/password), used only for Instructor accounts.

#### Storage
Firebase Cloud Storage, for optional Instructor-uploaded Assignment attachments (e.g., a rubric PDF), capped at 5MB. (Assumption: size cap carried over from the original architecture discussion; team should confirm.)
Upload flow: the frontend uploads the file directly to Firebase Storage via the client SDK first and receives back a download URL. Only that URL is then included as attachmentUrl in the REST API call (API-05 / API-08) to Cloud Functions — the file itself never passes through the Cloud Function. This is the one intentional exception to "all data access goes through the REST API," since attachment bytes are not Firestore data and Storage has its own security rules.

#### External Services
None in MVP. The system is entirely self-contained within the Instructor-facing application; there are no outbound integrations, webhooks, or third-party AI calls.

---

## 10. Technology Stack

| Layer | Technology | Reason |
|---|---|---|
| Frontend | React | Team-specified; fast to build the 3 core instructor screens |
| Backend | Firebase SDK / Cloud Functions | Team-specified; Functions expose the single authenticated REST API that handles all Instructor CRUD (Cohorts, Assignments) |
| Database | Firestore | Team-specified NoSQL store; a good fit since the MVP data model is simple (User → Cohort → Assignment) with no deep relational joins |
| Auth | Firebase Auth | Team-specified; fast, free-tier email/password auth for Instructors |
| Hosting | Firebase Hosting | Team-specified; free static hosting integrated with the rest of Firebase |

---

## 11. API / Interfaces

All REST endpoints are exposed as HTTPS-triggered Firebase Cloud Functions, require a valid Firebase ID token, and are documented as a shared contract (e.g., OpenAPI/Postman collection).

### API-01 — `POST /cohorts`
Create a Cohort. Owning Instructor only.

### API-02 — `GET /cohorts`
List the authenticated Instructor's own Cohorts.

### API-03 — `PUT /cohorts/:id`
Rename a Cohort. Owning Instructor only.

### API-04 — `DELETE /cohorts/:id`
Delete a Cohort. Owning Instructor only.

### API-05 — `POST /assignments`
Create an Assignment. Body: {cohortId, title, description, criteria, dueDate, attachmentUrl?}. attachmentUrl is optional and, if present, must already point to a file the Instructor has uploaded to Firebase Storage (see Section 9, Storage — upload happens client-side before this call). Owning Instructor only.

### API-06 — `GET /assignments?cohortId=`
List the Assignments within one of the authenticated Instructor's own Cohorts.

### API-07 — `GET /assignments/:id`
Fetch a single Assignment by ID. Owning Instructor only. Used by the Edit Assignment screen to load current values.

### API-08 — `PUT /assignments/:id`
Update an Assignment. Same optional attachmentUrl handling as API-05 (upload to Storage first, then send the URL). Owning Instructor only.

### API-09 — `DELETE /assignments/:id`
Delete an Assignment. Owning Instructor only.

---

## 12. Security

### Authentication
Firebase Auth (email/password) issues ID tokens; every Cloud Function REST endpoint verifies the Firebase ID token before processing a request. There is no unauthenticated endpoint in MVP.

### Authorization
Since the frontend has no direct Firestore access (Section 9), ownership checks live in one place:
- Cloud Function checks on every REST endpoint (API-01 through API-09) re-validate ownership server-side before any read or write (e.g., a request against a Cohort/Assignment only succeeds if InstructorID == request.auth.uid for the token presented).
- Firestore Security Rules are set to deny all direct client reads/writes to the cohorts and assignments collections. This is a defense-in-depth backstop, not the primary authorization layer — even if a client somehow bypassed the REST API, the rules block it. Only the Cloud Functions service account (via the Admin SDK) can read/write these collections.
- Storage Security Rules separately restrict attachment access to the owning Instructor, since file upload/download goes directly through the client SDK rather than the REST API (Section 9).

### Data Protection
- No password or password hash is ever stored in Firestore or exposed via any read path — Firebase Auth owns that data entirely (Section 8).
- No endpoint returns another Instructor's data.
- Uploaded attachments are size-capped at 5MB and access-controlled via Storage security rules tied to the owning Instructor.

---

## 13. Error Handling

### Expected Errors
- Invalid/expired/missing auth token on any endpoint → 401.
- Attempt to read/write another Instructor's data → 403.
- Missing required fields (e.g., no due date) → 400 with field-level validation messages.
- Request for a Cohort/Assignment ID that doesn't exist or isn't owned by the requester → 404.

### Failure Scenarios
- **File upload failure** (size/type) → reject client-side and server-side (Storage rules) with a clear error message.

---

## 14. Deployment

### Development
- Firebase Emulator Suite (Auth, Firestore, Functions, Storage) for local development and testing, avoiding any cost during dev.
- Feature branches with PR review before merging to `main`.

### Production
- Firebase Hosting for the React frontend (`firebase deploy`).
- Firebase Cloud Functions deployed via Firebase CLI on the **Blaze (pay-as-you-go)** plan — see CON-01 and Risk R-01, since Functions require Blaze even though usage can stay within the free-tier quota.
- Firestore/Auth/Storage remain on the free tier.

---

## 15. Constraints

- **Budget:** $0 deployment budget (CON-01).
- **Time:** One semester.
- **Team:** 3–5 student developers.
- **Free Tier:** All services must operate within free-tier quotas; any feature requiring a paid tier (e.g., Cloud Functions' Blaze plan) must be flagged and mitigated (see Risks).

---

## 16. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Firebase Cloud Functions require the Blaze plan, which requires entering a credit card — in tension with the zero-budget constraint (CON-01) | High — could block the REST API layer entirely if students won't add a card | Confirm upfront whether students can add a card to Blaze (usage can still be $0 with budget alerts/quotas); if not, move the Functions layer to a free alternative (e.g., a free-tier Node server) while keeping Firestore/Auth/Storage on Firebase |
| Bugs in Firestore/Storage Security Rules | High — could expose one Instructor's data to another | Write and run automated rules tests (contributes to the 7+ test requirement) against the Firebase Emulator before each deploy |
| Small team / one semester timeline | Medium — scope creep could prevent MVP completion | Strict adherence to Section 2 scope; defer anything not listed to Section 18 |

---

## 17. Acceptance Criteria

### MVP is complete when:

- [ ] Instructors can sign up, log in, create a Cohort, and create an Assignment with a due date and criteria.
- [ ] The 3 core screens (Login, Dashboard, Create/Edit Assignment) are implemented and functional.
- [ ] Instructors can view, edit, and delete their own Cohorts and Assignments from the Dashboard.
- [ ] At least 6 REST API operations are implemented and documented in a shared contract (OpenAPI or equivalent), and every operation requires authentication.
- [ ] At least 7 automated tests cover essential paths (auth, Cohort/Assignment CRUD, and security-rule enforcement for cross-instructor isolation).
- [ ] Firestore/Storage Security Rules prevent one Instructor from reading/writing another Instructor's data (verified by tests).
- [ ] The entire system runs within free-tier quotas (or an agreed, explicitly-approved Blaze-plan usage with $0 actual spend).

---

## 18. Future Improvements

- **Read-only, link-based sharing of a Cohort's Assignments** (no login required) so students can view due dates and criteria directly — deferred out of MVP to keep the system purely instructor-facing at this stage.
- **Inter-team webhook integration**, including an AI-assisted summary step with a deterministic fallback, for notifying other systems when an Assignment is created — deferred out of MVP.
- **Student accounts, self-enrollment (via join code), and a full submission workflow** (file upload, tracking, instructor review) — this was part of the original data model concept but is deferred out of MVP now that Instructors are the sole target user.
- Notification system (email/push) for upcoming due dates.
- Instructor gradebook/analytics dashboard.
- Native mobile app.
- Plagiarism detection integration.
- In-app messaging between instructors and students (replacing WhatsApp/Discord use).
- Bulk CSV import for Cohort rosters.
