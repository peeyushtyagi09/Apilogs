 you have to one this that is read utils foldeer in place of controllers folder and complete utils.md in place of controllers.md file - instruction -> # APILogs Backend — Routes Documentation

>You are documenting the **APILogs backend** for a developer who needs to understand the complete project for technical interviews.

The existing documentation is in:

`backend/src/controllers/README.md`

Your task is to thoroughly inspect the actual source code and update this README so that it accurately explains what is happening inside every controller.

### 1. Files to inspect

Read all files in:

`backend/src/controllers/`

At minimum, inspect:

* `auth.Controller.js`
* `event.controller.js`
* `incident.controller.js`
* `project.Controller.js`
* `system.controller.js`

Also inspect every dependency required to understand their actual behavior:

* Routes that call these controllers
* Authentication and API-key middleware
* All relevant Mongoose models
* Utility functions
* JWT and OTP generation/verification code
* Metrics implementation
* Socket.IO server and socket manager
* Configuration and environment-variable usage
* Relevant schemas, validators, and database indexes

Follow imports and function calls into other files. Do not stop at the controller files.

### 2. Documentation requirements

Update `backend/src/controllers/README.md` using the existing structure, but make it substantially more complete.

For every controller file, document:

1. **Purpose:** What responsibility the controller has in the application.
2. **Dependencies:** Every imported module and exactly how it is used.
3. **Routes:** HTTP method, endpoint, route parameters, query parameters, request body, and middleware chain. Verify these from the actual route files.
4. **Authentication and authorization:** Explain which middleware runs, what it validates, and how ownership or permissions are checked.
5. **Internal helper functions:** Explain every helper, its inputs, outputs, and purpose.
6. **Every exported function:** Document the actual implementation step by step, in execution order.
7. **Database operations:** Explain the models used, fields read or written, queries, updates, population, sorting, pagination, and relevant indexes.
8. **Validation:** Explain every validation rule, default value, allowed enum, and edge case present in the code.
9. **Error handling:** Document every meaningful error branch, status code, response body, and whether errors are logged, propagated, or swallowed.
10. **Security:** Explain actual authentication, authorization, hashing, token handling, rate limiting, sanitization, and security protections. Distinguish implemented protections from merely intended protections.
11. **Side effects:** Explain email delivery, metrics changes, audit logs, WebSocket broadcasts, incident creation, and other side effects.
12. **Return values:** Show the actual response structure and important fields.
13. **Interview explanation:** Add a short explanation of how a developer should explain this function in an interview.

### 3. Trace the complete execution flow

For important endpoints, especially:

* User registration and email verification
* Password login
* OTP login
* JWT refresh
* Logout everywhere
* Project creation and API-key rotation
* Event ingestion
* Automatic incident creation or updating
* Event retrieval and pagination
* Incident status updates
* Health and metrics endpoints

Explain the full request lifecycle:

`Client → Route → Middleware → Controller → Model/Utility → Database or external service → WebSocket/side effect → HTTP response`

Use the actual names of files, functions, fields, and variables from the code.

### 4. Explain important technical details

Do not merely repeat the existing README. Verify and explain:

* How JWT access and refresh tokens are generated and validated.
* How token versioning invalidates sessions.
* How OTPs are generated, hashed, stored, expired, consumed, and limited by attempts.
* How API keys are generated, stored, hashed, selected, rotated, and verified.
* How multi-tenancy and project ownership are enforced.
* How event severity triggers incident handling.
* How incident grouping and `messageSignature` work.
* How Socket.IO rooms and event broadcasts work.
* How MongoDB indexes support the queries.
* How cursor-based pagination actually works.
* How metrics are stored and updated.
* How errors and partial failures are handled.
* Any race conditions, security weaknesses, missing validations, or implementation risks you discover.

### 5. Verify all claims

This is critical.

* Treat the source code as the source of truth.
* Do not assume the existing README is correct.
* Do not invent functionality, performance results, security guarantees, or test results.
* Do not claim that a feature is implemented unless you find evidence in the code.
* If the README and code disagree, document the actual behavior and explicitly add a section called **Documentation discrepancies**.
* If something cannot be verified, label it **Not verified from the available source code**.
* If the CV claims a performance metric, do not treat it as proven unless benchmarks or measurement code support it.
* Identify any discrepancy between SHA-256 API-key hashing in the CV and the hashing mechanism actually used by the project.

### 6. Improve the README structure

Keep the existing five-controller organization, but add:

* A controller architecture overview
* A route and middleware map
* A complete endpoint summary table
* Detailed documentation for every exported function
* Request lifecycle diagrams using Mermaid where useful
* Database and model dependency relationships
* Authentication and authorization flow
* Real-time event and incident flow
* Error-handling conventions
* Security implementation notes
* Performance and pagination notes
* Documentation discrepancies
* Unverified assumptions
* A final **Interview Preparation Notes** section with likely technical questions and the exact implementation concepts needed to answer them

### 7. Quality checks

Before finishing:

* Confirm that every controller file has been read.
* Confirm that every exported function has been documented.
* Confirm that route paths and middleware are verified from source.
* Confirm that all model and utility dependencies mentioned are actually used.
* Check for inconsistencies in status codes, field names, hashing algorithms, authentication requirements, and response formats.
* Preserve correct existing documentation where it matches the source.
* Do not modify application source code. Only update the documentation file.

At the end, provide a concise report containing:

1. Files inspected.
2. Documentation sections added or updated.
3. Discrepancies found between the old README and source code.
4. Important implementation details that were previously missing.
5. Remaining areas that could not be verified.
