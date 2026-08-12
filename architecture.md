## Architecture

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