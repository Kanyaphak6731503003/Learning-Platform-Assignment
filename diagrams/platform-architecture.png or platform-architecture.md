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