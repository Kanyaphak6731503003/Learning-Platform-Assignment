
###  Product Manager Review 

Overall, this is a highly pragmatic and well-scoped Product Requirements Document (PRD). The decision to ruthlessly prioritize the Instructor's experience and cut the student-facing features for the MVP shows a strong understanding of constraints (0$ budget, 1-semester timeline, small student team).

Here is the assessment based on your key questions:

* **Is the problem specific and supported by a clear target user?**
Yes. The problem (scattered assignment details) is well-defined, and restricting the target user exclusively to "Instructors" is a smart product decision that drastically simplifies the MVP.


* **Does the MVP solve the core journey before adding extra features?**
Absolutely. The core journey focuses entirely on creating a single source of truth (Cohorts and Assignments). All secondary features (student submissions, AI summaries, notifications) are appropriately moved to Out of Scope / Future Improvements.


* **Are priorities and out-of-scope items explicit?**
Yes. Section 2 clearly separates In Scope from Out of Scope, leaving no ambiguity for the engineering team regarding what *not* to build.


* **Are requirements testable and acceptance criteria measurable?**
Mostly yes. The reliance on the Firebase Client SDK without a REST API is explicit. However, some Non-Functional Requirements and Acceptances Criteria could be tightened for better testability (see findings below).


* **Do business rules reflect real product decisions?**
Yes. BR-01 to BR-03 directly address data isolation and access control, which are the most critical risks in a 100% BaaS architecture.



**Specific PM Findings:**

* PM-01 — NFR-01 specifies a load time of "under 2 seconds on a typical broadband connection". "Typical broadband" is slightly ambiguous. We should define specific bandwidth/latency metrics (e.g., "under 2 seconds on a 10 Mbps connection") to make this strictly testable.


* PM-02 — FR-03 mentions an "optional file attachment" for assignments. While Section 9 mentions a 5MB limit, the FR does not specify allowed file types (e.g., PDF, DOCX, JPG). We should explicitly define supported MIME types to prevent users from uploading malicious scripts or unsupported files.


* PM-03 — The Acceptance Criteria requires "At least 7 automated tests". While measurable, tying the AC to an arbitrary number of tests might encourage quantity over quality. It would be better to state that "Automated tests must cover 100% of the Firestore Security Rules paths outlined in Section 12."


* PM-04 — BR-01 states "Public sign-up creates an Instructor account". While this removes friction for MVP, allowing unrestricted public sign-ups could expose the system to bot attacks that exhaust the Firebase Spark plan free-tier limits (CON-01). We might need a simple CAPTCHA or an invite-only mechanism if this becomes a public URL.


* PM-05 — DA-04 mentions "Frontend or Rules must ensure no orphaned assignments remain" when a Cohort is deleted. This is an architectural ambiguity. The PRD should explicitly decide *how* this is handled (e.g., "The frontend must execute a Firestore Batched Write to delete all child assignments before deleting the Cohort document").


###  Frontend UX/UI Designer Review 

Overall, the PRD provides a solid technical and functional foundation by clearly defining the 3 core screens (Login, Home, Create/Edit Assignment). However, from a UX/UI perspective, the document focuses heavily on database operations and backend security, leaving several critical user interface states and interactive behaviors undefined.

Here is the assessment based on your key questions:

* **Can users complete the core journey with understandable screens and states?**
Yes, the 3 core screens cover the basic flow. However, the navigation between these screens (e.g., how to go back from 'Edit Assignment' to 'Home') is not explicitly detailed.


* **Are empty, loading, error, success, and permission-denied states identified?**
Only partially. The PRD does a good job identifying backend error scenarios (e.g., `permission-denied`, file size limits). However, UI-specific states like loading indicators, empty states, and success messages are missing.


* **Is the interface usable on relevant screen sizes?**
The PRD explicitly excludes native mobile apps, but it does not specify whether the React web frontend needs to be responsive for mobile or tablet browsers.


* **Are form validation, feedback, and accessibility considered?**
Backend validation is covered via Security Rules, but frontend validation (e.g., checking if a due date is in the past before submitting) and accessibility standards (a11y) are not mentioned.


* **Could a user make an irreversible or confusing mistake?**
Yes. Deleting a cohort or assignment is listed as a feature, but there are no safeguards mentioned to prevent accidental data loss.



**Specific UX/UI Findings:**

* **UX-01 — Empty States:** The PRD mentions fetching and listing Cohorts and Assignments (DA-02, DA-06), but it lacks design requirements for "Empty States" (e.g., what the Home screen looks like when a new Instructor logs in and has zero Cohorts).


* **UX-02 — Destructive Actions:** DA-04 and DA-09 allow deleting Cohorts and Assignments. The PRD does not specify if the UI requires a confirmation modal (e.g., "Are you sure you want to delete this Cohort and all its assignments?") to prevent irreversible mistakes.


* **UX-03 — Loading and Success Feedback:** While Section 13 covers Error Handling, there is no mention of loading states (e.g., skeleton loaders or spinners during the < 2 seconds load time mentioned in NFR-01) or success states (e.g., a toast notification saying "Assignment Created").


* **UX-04 — Frontend Form Validation:** The PRD mentions intercepting missing fields via Security Rules. Relying solely on the database for validation creates a poor UX. We need frontend validation requirements (e.g., disabling the "Submit" button until the Title is filled, or showing inline errors if the uploaded attachment exceeds the 5MB limit mentioned in Section 9).


* **UX-05 — Responsive Design:** While "Native mobile apps" are Out of Scope, instructors may still access the web app via mobile browsers. The PRD should clarify if the Tailwind CSS implementation must be fully responsive or if it is strictly designed for desktop use.


###  Backend API and Database Review 

Overall, this PRD takes a pragmatic approach by adopting a 100% Backend-as-a-Service (BaaS) architecture using Firebase, which perfectly aligns with the $0 budget and tight timeline. Since there is no custom REST API layer (no Node.js or Cloud Functions), the "Backend Review" effectively evaluates the Firestore Data Model, SDK Data Access (DA) operations, and Firestore/Storage Security Rules.

Here is the assessment based on your key questions:

* **Is every requirement supported by a clear data owner and API responsibility?**
Yes. The Instructor is explicitly defined as the sole data owner. Because there is no traditional backend, the responsibility of data manipulation falls on the React frontend using the Firebase Client SDK, while the responsibility of access control falls entirely on Firestore Security Rules.


* **Are entity relationships and required fields sufficient?**
The relationships (User to Cohort to Assignment) are flat and well-suited for a NoSQL document database. However, while `AttachmentURL` is explicitly marked as optional, the PRD lacks strict definitions on which other fields are absolutely required versus nullable.


* **What prevents duplicates, invalid states, and race conditions?**
This is a weak point in the PRD. While it mentions using Firestore Batched Writes to prevent orphaned data, NoSQL databases like Firestore do not inherently enforce unique constraints on fields like "Cohort Name".


* **Which rules must be enforced on the backend rather than the frontend?**
The PRD correctly identifies that authorization, tenant isolation (cross-instructor access prevention), and schema validation must be enforced at the database layer via Security Rules. It also correctly utilizes Firebase App Check to prevent bot attacks.


* **Are API inputs, outputs, and error cases clear enough to implement?**
The Data Access operations (DA-01 through DA-09) clearly map out the required SDK commands (`addDoc`, `getDocs`, `updateDoc`, `deleteDoc`). Expected Firebase SDK error codes (`unauthenticated`, `permission-denied`, `not-found`) are also clearly identified.



**Specific Backend Findings:**

* BE-01 — DA-04 requires deleting a Cohort and ensuring no orphaned assignments remain. Firestore does not support automatic cascading deletes. The PRD mentions frontend Batched Writes, but if a user maliciously calls the `deleteDoc` API directly, it could leave orphaned assignment documents. The strategy for cascading deletes must be explicitly defined.


* BE-02 — The Data Model lists fields like `DueDate` for Assignments, but lacks data type specifics. To enforce schema validation in Security Rules (as mentioned in Section 13), we must define if `DueDate` is a Firestore Timestamp, a UTC ISO string, or an integer.


* BE-03 — Section 9 states file uploads are capped at 5MB and protected by Storage Security Rules. However, it does not specify which MIME types are allowed. Storage Security Rules must validate `request.resource.contentType` to prevent users from uploading executable files or malicious scripts.


* BE-04 — BR-01 allows public sign-up to create an Instructor account. While Firebase App Check mitigates bot scripts, a malicious user could still manually create thousands of accounts via the Auth API, potentially exhausting the free Spark plan limits. A rate-limiting strategy or reCAPTCHA requirement should be considered for the Auth layer.


###  Quality and Security Review 

Overall, the PRD demonstrates a strong foundational understanding of cloud security principles, specifically by leaning entirely on Firebase Authentication and Firestore Security Rules to secure the 100% BaaS architecture. The explicit separation of authentication (Firebase Auth) and authorization (Security Rules) is an excellent architectural decision for this tech stack. However, there are gaps in input validation, abuse prevention, and specific testing strategies.

Here is the assessment based on your key questions:

* **How will each acceptance criterion be tested?**
The PRD explicitly mandates using the Firebase Emulator Suite to run automated tests against Firestore and Storage rules before deployment. However, defining the requirement merely as "At least 7 automated tests" is insufficient for ensuring comprehensive quality and security coverage.


* **Are authentication and authorization clearly separated?**
Yes, this is handled very well. Authentication is delegated to Firebase Auth (issuing tokens), while Authorization is handled exclusively by Firestore/Storage Security Rules (checking `request.auth.uid`).


* **Can users access or change only permitted data?**
Yes, theoretically. BR-02 and NFR-02 explicitly enforce tenant isolation, ensuring instructors can only read/write their own data, backed by database-level security rules.


* **Are sensitive data, validation, logging, and secrets handled safely?**
Sensitive data (passwords) is handled safely as it is never stored in Firestore, only within Firebase Auth. However, while the PRD mentions schema validation intercepting invalid data types in Security Rules, the exact data types and required fields are not strictly defined in the Data Model. Logging is not mentioned.


* **What happens during network, database, and third-party service failures?**
Network failures are gracefully handled via the Firebase Client SDK's Offline Persistence, allowing instructors to save data locally during disruptions and sync later. Since the MVP has no third-party integrations, external service failures are not a risk.



**Specific QA & Security Findings:**

* **QS-01 — Threat to Availability (Spark Plan Limits):** BR-01 states public sign-up creates an Instructor account. While Firebase App Check protects against bot scripts, a motivated attacker could still manually or programmatically bypass it to spam account creation or generate massive payloads, quickly exhausting the $0 budget / Firebase Spark Plan limits (CON-01). Rate limiting or CAPTCHA at the Auth layer should be required.


* **QS-02 — Insufficient File Validation:** Section 9 caps file uploads at 5MB and protects them via Storage Rules. However, it completely omits MIME-type validation. An attacker could upload malicious executables (e.g., `.exe`, `.sh`) disguised as attachments, creating a distribution vector for malware. Storage rules must strictly enforce allowed file types (e.g., `application/pdf`).


* **QS-03 — Vague Testing Requirements:** The Acceptance Criteria requires "At least 7 automated tests". Security testing should be based on coverage (e.g., "100% of Security Rules paths are tested for both ALLOW and DENY states"), not an arbitrary number.


* **QS-04 — Undefined Schema for Validation:** Section 13 expects Security Rules to reject missing required fields or invalid data types (`permission-denied`). However, the Data Model (Section 8) does not define which fields are strictly required (besides ID) or their specific types (e.g., is `DueDate` a string or a timestamp?). The QA team cannot write valid tests without a strict schema definition.


###  Delivery and Document Review 

Overall, this PRD is exceptionally well-aligned with its delivery constraints. By deliberately choosing a 100% BaaS architecture (Firebase) and cutting out a custom REST API, the product significantly increases its chances of being successfully delivered by a small student team (3-5 people) within the single-semester timeline and $0 budget constraints.

Here is the assessment based on your key questions:

* **Can the group build, deploy, test, and document this within one semester?**
Yes. Eliminating backend development and focusing purely on a React frontend connected directly to Firebase drastically reduces the workload and avoids the complexity of maintaining two separate codebases.


* **Are hosting, domain, environment variables, free-tier limits, and deployment steps realistic?**
The use of Firebase Hosting and the Spark Plan free-tier is highly realistic for a zero-budget student project. However, the PRD lacks details on environment variable management and specific deployment pipelines (see findings below).


* **Are dependencies, risks, and ownership visible?**
Yes, dependencies are consolidated entirely onto the Firebase ecosystem. Section 16 clearly identifies the primary technical risk (flawed security rules exposing data) and the primary project management risk (scope creep).


* **Is the PRD internally consistent and understandable by a new team member?**
The document is highly consistent. The narrative firmly sticks to the "Instructor-only" MVP scope and constantly reinforces the 100% BaaS approach without contradicting itself by suddenly introducing backend endpoints.


* **Is there a minimum release plan and a rollback or recovery approach where appropriate?**
This is a weak point. While Section 14 mentions seamless deployment via the Firebase CLI, there is no formal release plan, CI/CD pipeline definition, or rollback strategy in case of a critical failure.



**Specific Delivery & Document Findings:**

* **DD-01 — CI/CD and Rollback Strategy:** Section 14 states that production rules and hosting are deployed via Firebase CLI. It does not specify if this is a manual process run from a developer's local machine or automated (e.g., GitHub Actions). Furthermore, there is no rollback strategy defined if a flawed Firestore Security Rule is accidentally deployed to production.


* **DD-02 — Environment Variable Management:** The PRD mentions using the Firebase Emulator Suite for local development to avoid costs. However, there is no documented requirement on how the engineering team should manage environment variables (`.env`) to safely toggle the React frontend between the local emulator and the production environment.


* **DD-03 — Free-Tier Monitoring:** The project is strictly constrained to a $0 budget on the Firebase Spark Plan (CON-01). While this is realistic, the PRD does not require the setup of budget alerts or usage monitoring. A runaway frontend loop or a minor bot attack could exhaust the quota, bringing development or production to a halt.


* **DD-04 — Database Indexing Documentation:** The PRD mentions querying assignments where `CohortID == targetCohort` (DA-06). Firestore often requires composite indexes for specific queries. The PRD should mandate that the `firestore.indexes.json` file be documented and committed to the repository to ensure all developers have the same database configuration.

