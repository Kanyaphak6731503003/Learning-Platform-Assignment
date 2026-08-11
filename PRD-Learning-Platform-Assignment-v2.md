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
- **Direct database integrations using the Firebase Client SDK for all core CRUD operations, protected by Security Rules.**
- At least 7 automated tests covering essential paths and validation.

### Out of Scope (MVP)
- Student accounts, login, or self-enrollment.
- Any public or unauthenticated access to assignment data (no share links, no read-only views).
- Submission workflow (students uploading/submitting work) - moved to Future Improvements.
- Inter-team webhook integrations and AI-assisted features - moved to Future Improvements.
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
The system architecture adopts a 100% **Backend-as-a-Service (BaaS)** approach using Firebase. This directly addresses the constraints of a zero-dollar budget ($0) and a one-semester development timeline for a small student team.

There will be no custom REST API layer (no Node.js or Cloud Functions). Instead, the React frontend will connect, communicate, and perform all CRUD operations directly against the database using the **Firebase Client SDK**. All authorization and business logic will be handled centrally by **Firestore Security Rules**. This architecture eliminates server cold starts, reduces the burden of maintaining two separate codebases, and ensures the project remains entirely within the free-tier limits of the Firebase Spark Plan.

### Architecture Diagram
```text
[ Instructor's React App (Frontend) ]
       |      |      |
       |      |      |-- (1) Authenticate --> [ Firebase Auth ] (Email/Password)
       |      |
       |      |--------- (2) Direct Upload -> [ Firebase Storage ] (Files/Attachments)
       |                                          ^ (Protected by Storage Security Rules)
       |
       |---------------- (3) Data CRUD -----> [ Cloud Firestore DB ] (NoSQL)
                                                  ^ (Protected by Firestore Security Rules)
                                                  ^ (Secured by Firebase App Check)

```

### Components

#### Frontend

* **Technology:** React (Vite or Create React App) + Tailwind CSS.
* **Role:** Serves as the user interface (UI) for Instructors and manages the primary data flow. It uses the Firebase Client JS SDK to read and write data directly (including executing Firestore Batched Writes to prevent orphaned data) and manages the application state.
* **Hosting:** Deployed on Firebase Hosting.

#### Backend

* **Technology:** Serverless BaaS (Firebase Security Rules + Firebase App Check).
* **Role:** There is no custom backend (REST API) in this MVP. The traditional backend role is entirely replaced by Firestore Security Rules, which run on Google's servers. These rules enforce strict authorization, prevent cross-cohort data access (Tenant Isolation), and validate incoming data schemas. Additionally, Firebase App Check is implemented to protect the system from bot attacks or malicious external scripts attempting to exhaust database quotas.

#### Database

* **Technology:** Cloud Firestore (NoSQL Document Database).
* **Role:** Stores Users (Instructors), Cohorts, and Assignments. The data structure is designed to be flat, making it an ideal fit for the NoSQL document model.
* **Feature:** Offline Persistence is enabled via the Client SDK, allowing Instructors to use the app and save data even during network disruptions. The data will automatically sync to the cloud once the connection is restored.

#### Authentication

* **Technology:** Firebase Authentication.
* **Role:** Manages the Sign-Up and Log-In process via Email and Password for Instructors only (there are no Student accounts in the MVP scope). It automatically issues and refreshes ID tokens used to securely authenticate requests against Firestore and Storage.

#### Storage

* **Technology:** Firebase Cloud Storage.
* **Role:** Used solely for storing Assignment attachments (e.g., rubrics or instruction PDFs). The frontend directly uploads files to Storage and then saves the resulting download URL into the Firestore document.
* **Constraint:** File uploads are capped at 5MB per file and are strictly protected by Storage Security Rules, ensuring files are accessible only to the Instructor who owns them.

#### External Services

* **Technology:** None (in MVP).
* **Role:** The system is entirely self-contained. There are no connections to third-party APIs, external webhooks, or AI services in the MVP phase. This intentionally reduces complexity and ensures the project scope can be realistically completed within one semester.

---

## 10. Technology Stack

| Layer | Technology | Reason |
| --- | --- | --- |
| **Frontend** | React | Team-specified; fast to build the 3 core instructor screens |
| **Backend** | Firebase Security Rules & App Check | Replaces the traditional REST API; enforces all business logic, data validation, and authorization directly at the database layer |
| **Database** | Firestore | Team-specified NoSQL store; a good fit since the MVP data model is simple (User → Cohort → Assignment) |
| **Auth** | Firebase Auth | Team-specified; fast, free-tier email/password auth for Instructors |
| **Hosting** | Firebase Hosting | Team-specified; free static hosting integrated with the rest of Firebase |

---

## 11. Data Access / SDK Operations

Since the architecture is 100% BaaS, there are no REST API endpoints. Instead, the React frontend executes CRUD operations directly against Firestore via the Firebase Client SDK. All operations require a valid authenticated session.

* **DA-01 - Create Cohort:** `addDoc` to the `cohorts` collection. Owning Instructor only.
* **DA-02 - List Cohorts:** `getDocs` from `cohorts` where `InstructorID == currentUser.uid`.
* **DA-03 - Rename Cohort:** `updateDoc` on a specific `cohorts` document. Owning Instructor only.
* **DA-04 - Delete Cohort:** `deleteDoc` on a specific `cohorts` document. (Frontend or Rules must ensure no orphaned assignments remain).
* **DA-05 - Create Assignment:** `addDoc` to the `assignments` collection. Includes optional `attachmentUrl` generated from a prior client-side Firebase Storage upload.
* **DA-06 - List Assignments:** `getDocs` from `assignments` where `CohortID == targetCohort`.
* **DA-07 - Fetch Assignment:** `getDoc` on a specific `assignments` document for the Edit screen.
* **DA-08 - Update Assignment:** `updateDoc` on a specific `assignments` document.
* **DA-09 - Delete Assignment:** `deleteDoc` on a specific `assignments` document.

---

## 12. Security

### Authentication

Firebase Auth (email/password) manages user sessions. The Firebase Client SDK automatically passes the user's Auth token with every database and storage request.

### Authorization

Since the frontend accesses Firestore directly, ownership checks live entirely in **Firestore Security Rules**.

* Rules validate every request against the authenticated user (e.g., `allow read, write: if request.auth != null && request.auth.uid == resource.data.InstructorID;`).
* There is no server-side middleware. If a client attempts an unauthorized operation or modifies the frontend code to access another instructor's data, the Firestore Rules act as an impenetrable wall and reject the request.
* Storage Security Rules separately restrict attachment upload/download access to the owning Instructor.

### Data Protection

* No password or password hash is ever stored in Firestore or exposed via any read path. Firebase Auth owns that data entirely.
* No database query returns another Instructor's data.
* Uploaded attachments are size-capped at 5MB and access-controlled via Storage security rules tied to the owning Instructor.

---

## 13. Error Handling

### Expected Errors (Firebase SDK Codes)

* Unauthenticated access → `unauthenticated`
* Attempt to read/write another Instructor's data → `permission-denied`
* Missing required fields or invalid data types → `permission-denied` (intercepted by schema validation in Security Rules)
* Document doesn't exist → `not-found`

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
* **The entire production environment runs strictly on the free Spark plan.**

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
| **Bugs in Firestore/Storage Security Rules** | **High** - Since there is no backend API, a flawed rule could expose one Instructor's data to another, or allow destructive unauthorized writes. | Strict adherence to writing and running automated rules tests against the Firebase Emulator before every deploy. |
| **Small team / one semester timeline** | **Medium** - Scope creep could prevent MVP completion. | Strict adherence to Section 2 scope; defer anything not listed to Section 18. |

---

## 17. Acceptance Criteria

### MVP is complete when:

* [ ] Instructors can sign up, log in, create a Cohort, and create an Assignment with a due date and criteria.
* [ ] The 3 core screens (Login, Home, Create/Edit Assignment) are implemented and functional.
* [ ] Instructors can view, edit, and delete their own Cohorts and Assignments from the Home.
* **[ ] The React frontend successfully executes data operations directly via the Firebase Client SDK without relying on a custom REST API.**
* [ ] At least 7 automated tests cover essential paths (auth, Cohort/Assignment CRUD, and security-rule enforcement for cross-instructor isolation).
* [ ] Firestore/Storage Security Rules prevent one Instructor from reading/writing another Instructor's data (verified by tests).
* **[ ] The entire system runs successfully on the Firebase Spark (Free) Plan with a $0 deployment cost.**

---

## 18. Future Improvements

* **Read-only, link-based sharing of a Cohort's Assignments** (no login required) so students can view due dates and criteria directly - deferred out of MVP to keep the system purely instructor-facing at this stage.
* **Inter-team webhook integration**, including an AI-assisted summary step with a deterministic fallback, for notifying other systems when an Assignment is created - deferred out of MVP.
* **Student accounts, self-enrollment (via join code), and a full submission workflow** (file upload, tracking, instructor review) - this was part of the original data model concept but is deferred out of MVP now that Instructors are the sole target user.
* Notification system (email/push) for upcoming due dates.
* Instructor gradebook/analytics Home.
* Native mobile app.
* Plagiarism detection integration.
* In-app messaging between instructors and students (replacing WhatsApp/Discord use).
* Bulk CSV import for Cohort rosters.