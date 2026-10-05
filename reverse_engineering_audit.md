# Reverse-Engineering Exercise & Security/Logic Audit: Espionage Event Platform

## Stage 0 — Recon

### 1. Application Overview
The application is a full-lifecycle hackathon/competition management platform named **"ESPIONAGE"** (package name: `dbug-espionage`), built for orchestrating technical competitions, participant admissions, on-site check-ins, automated assessments, and live sandboxed coding rounds.

### 2. Main Components
- **Registration & Verification Portal (`/register`, `/success`)**: Participant sign-up flow supporting solo and duo entries, Cloudflare Turnstile CAPTCHA validation, Nodemailer email OTP verification, and auto-generated QR entry passes.
- **Participant Access & Dashboard (`/login`, `/dashboard`)**: OTP-based login for confirmed team leaders, live stage status monitoring, and participant progress tracking.
- **Competition Engines**:
  - **Round 1 (`/api/round1/*`)**: Timed online aptitude/technical MCQ evaluation with scoring and client anti-cheat tracking.
  - **Round 2 (`/api/round2/*`)**: Live coding challenge interface using embedded Monaco Editor, integrated with the Piston sandbox execution engine and OpenRouter LLM automated evaluator.
- **Event Operations & Physical Attendance (`/attendance`, `/oc`)**: Volunteer/organizer scanning portal utilizing web cameras (`html5-qrcode`) to record on-site attendance for Round 1 and Round 2.
- **Admin Control Center (`/admin`, `/api/admin/*`)**: Event state toggling (`EventConfig`), question database seeding, automated top-N shortlisting, score exports, and certificate compilation (`pdf-lib`, `jszip`).

### 3. Technology Stack
- **Framework**: Next.js 16.2.1 (App Router), React 19, TypeScript 5
- **Styling**: Tailwind CSS 4
- **Database**: MongoDB managed via Mongoose 9.2.4
- **Sandboxing & AI**: Piston API execution service (`src/lib/piston.ts`), OpenRouter AI SDK
- **Utilities & Communications**: Nodemailer (SMTP), Cloudflare Turnstile CAPTCHA, `pdf-lib`, `jszip`, `html5-qrcode`, `qrcode`

### 4. Important Directory Structure
- `src/app/api/`: REST API handlers categorized into `admin`, `attendance`, `auth`, `dashboard`, `round1`, and `round2`.
- `src/app/(screens)/`: Page layouts and interactive views (`/register`, `/dashboard`, `/admin`, `/attendance`, `/rsvp`).
- `src/models/`: Mongoose schemas defining system records (`Participant`, `CodingQuestion`, `MCQQuestion`, `EventConfig`, `OTP`, `Team`, `Organizer`, `Notification`).
- `src/lib/`: Backend utilities for database connectivity (`mongodb.ts`), email dispatch (`mailer.ts`), rate limiting (`rate-limit.ts`), code execution (`piston.ts`), and stage locks (`accessControl.ts`).

---

## Stage 3 — Server-Side Routes

The following table documents the core server-side API routes discovered directly from file handlers in `src/app/api/**/route.ts`:

| Method | Path | Purpose | Inputs | Outputs | Authentication & Authorization | File:Line Evidence |
|---|---|---|---|---|---|---|
| `POST` | `/api/send-otp` | Sends email OTP during registration | `{ email, turnstileToken }` | `{ success: true, message: string }` | Public; verifies Cloudflare Turnstile token | [src/app/api/send-otp/route.ts:22](file:///Users/tanushrid/hackback/espionage-event/src/app/api/send-otp/route.ts#L22) |
| `POST` | `/api/verify-otp` | Verifies registration OTP | `{ email, otp }` | `{ success: true, message: string }` | Public; verifies OTP matching unverified record in DB | [src/app/api/verify-otp/route.ts:8](file:///Users/tanushrid/hackback/espionage-event/src/app/api/verify-otp/route.ts#L8) |
| `POST` | `/api/register-manual` | Registers participant or team | `{ name, email, phone, college, department, year, rollNumber, teamType, partner? }` | `{ success: true, participantId: string }` | None; unauthenticated endpoint | [src/app/api/register-manual/route.ts:7](file:///Users/tanushrid/hackback/espionage-event/src/app/api/register-manual/route.ts#L7) |
| `POST` | `/api/auth/send-otp` | Sends login OTP to registered team leader | `{ email }` | `{ success: true, message: string }` | None; validates presence of confirmed RSVP in DB | [src/app/api/auth/send-otp/route.ts:34-43](file:///Users/tanushrid/hackback/espionage-event/src/app/api/auth/send-otp/route.ts#L34-L43) |
| `POST` | `/api/auth/verify-otp` | Validates login OTP and issues token | `{ email, otp }` | `{ success: true, session: { ...token } }` | Validates OTP against DB; generates ephemeral token | [src/app/api/auth/verify-otp/route.ts:21-61](file:///Users/tanushrid/hackback/espionage-event/src/app/api/auth/verify-otp/route.ts#L21-L61) |
| `GET` | `/api/dashboard/me` | Fetches participant score and state | Query string `?email=...` | `{ participant: {...} }` | **None found**; queryable by anyone knowing an email | [src/app/api/dashboard/me/route.ts:7-22](file:///Users/tanushrid/hackback/espionage-event/src/app/api/dashboard/me/route.ts#L7-L22) |
| `POST` | `/api/round1/submit` | Submits answers for Round 1 assessment | `{ email, answers, warnings, keyViolations }` | `{ success: true, score, correct, total }` | Checks `config.round1Active`; **no session token auth** | [src/app/api/round1/submit/route.ts:9-27](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round1/submit/route.ts#L9-L27) |
| `POST` | `/api/round2/execute` | Submits code to Piston runner for grading | `{ email, code, language, questionId }` | `{ verdict, passed, total, totalScore, results }` | Checks `round2Active` & `isShortlisted`; **no session token** | [src/app/api/round2/execute/route.ts:18-31](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round2/execute/route.ts#L18-L31) |
| `POST` | `/api/round2/submit` | Marks participant's Round 2 as finished | `{ email, warnings, keyViolations }` | `{ success: true, message: string }` | Checks `round2Active`; **no session token auth** | [src/app/api/round2/submit/route.ts:8-26](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round2/submit/route.ts#L8-L26) |
| `POST` | `/api/attendance` | Scans and marks attendee attendance | `{ participantId, round, secret }` | `{ success: true, participant: {...} }` | Secret comparison against `ORGANIZER_SECRET` | [src/app/api/attendance/route.ts:13-17](file:///Users/tanushrid/hackback/espionage-event/src/app/api/attendance/route.ts#L13-L17) |
| `POST` | `/api/admin/shortlist` | Auto-shortlists top N participants | `{ password, count }` | `{ success: true, shortlisted: number }` | Password comparison against `ADMIN_PASSWORD` | [src/app/api/admin/shortlist/route.ts:8-12](file:///Users/tanushrid/hackback/espionage-event/src/app/api/admin/shortlist/route.ts#L8-L12) |
| `DELETE` | `/api/admin/delete-participant` | Deletes a participant record | `{ password, participantId }` | `{ message: string }` | Password comparison against `ADMIN_PASSWORD` | [src/app/api/admin/delete-participant/route.ts:9-11](file:///Users/tanushrid/hackback/espionage-event/src/app/api/admin/delete-participant/route.ts#L9-L11) |

---

## Stage 4 — Data Model

Directly extracted from schemas defined in `src/models/*.ts`:

### 1. `Participant` ([src/models/Participant.ts:36-92](file:///Users/tanushrid/hackback/espionage-event/src/models/Participant.ts#L36-L92))
- **Purpose**: Primary entity tracking participant registration, academic metadata, competition scores, violations, attendance, and round qualifications.
- **Key Fields**:
  - `participantId`: `String`, unique index (`ESP-XXX`).
  - `name`, `email` (`String`, unique, lowercase index), `phone`, `college`, `department`, `year`, `rollNumber`.
  - `teamType`: `'solo' | 'duo'`, default `'solo'`.
  - `partner`: Embedded document containing `{ name, email, phone, college, department, year, rollNumber }`.
  - `rsvpStatus`: `'PENDING' | 'CONFIRMED' | 'DECLINED'`, default `'PENDING'`.
  - `attendanceRound1`, `attendanceRound2`: `Boolean`, default `false`.
  - `round1Score`: `Number`, `round1SubmittedAt`: `Date`.
  - `round1Warnings`, `round1KeyViolations`: `Number`, default `0`.
  - `isShortlisted`: `Boolean`, default `false`.
  - `round2Score`: `Number`, `round2SubmittedAt`: `Date`.
  - `round2Submissions`: Array of embedded subdocuments (`questionId`, `code`, `language`, `verdict`, `submittedAt`).
  - `round2FinalSubmissions`: Array of final submissions per question with test pass percentages.
- **Indexes**: Unique index on `participantId` and `email`.

### 2. `OTP` ([src/models/OTP.ts:3-15](file:///Users/tanushrid/hackback/espionage-event/src/models/OTP.ts#L3-L15))
- **Purpose**: Ephemeral storage for email verification codes.
- **Key Fields**: `email` (`String`, required, lowercase), `otp` (`String`, required), `expiresAt` (`Date`, required, TTL index expiring after 300 seconds), `verified` (`Boolean`, default `false`).
- **Indexes**: MongoDB TTL index on `expiresAt` (`{ expireAfterSeconds: 0 }`).

### 3. `CodingQuestion` ([src/models/CodingQuestion.ts:15-35](file:///Users/tanushrid/hackback/espionage-event/src/models/CodingQuestion.ts#L15-L35))
- **Purpose**: Challenge problems for Round 2 live coding.
- **Key Fields**: `title`, `description`, `difficulty` (`'easy' | 'medium' | 'hard'`), `points`, `order`, `timeLimit`, `memoryLimit`, `sampleInput`, `sampleOutput`, `hiddenTestCases` (Array of `{ input, expectedOutput }`).

### 4. `MCQQuestion` ([src/models/MCQQuestion.ts:10-24](file:///Users/tanushrid/hackback/espionage-event/src/models/MCQQuestion.ts#L10-L24))
- **Purpose**: Multiple-choice assessment questions for Round 1.
- **Key Fields**: `question`, `options` (`[String]`), `correctAnswer` (`Number`), `points`, `category`.

### 5. `EventConfig` ([src/models/EventConfig.ts:9-16](file:///Users/tanushrid/hackback/espionage-event/src/models/EventConfig.ts#L9-L16))
- **Purpose**: Singleton document controlling global competition state and stage transitions.
- **Key Fields**: `registrationOpen` (`Boolean`, default `true`), `round1Active` (`Boolean`, default `false`), `round2Active` (`Boolean`, default `false`).

### 6. `Team` ([src/models/Team.ts:22-53](file:///Users/tanushrid/hackback/espionage-event/src/models/Team.ts#L22-L53))
- **Purpose**: Standalone team grouping model with foreign key references to `Participant` (`members: [{ type: Schema.Types.ObjectId, ref: 'Participant' }]`). Largely bypassed in favor of embedded `partner` fields in `Participant`.

---

## Stage 7 — Gaps (Bugs, Security Weaknesses, and Logic Gaps)

1. **Authentication Token Disconnect**: In `/api/auth/verify-otp`, a random session token is generated (`crypto.randomBytes(32).toString('hex')`) and returned to the browser ([src/app/api/auth/verify-otp/route.ts:50](file:///Users/tanushrid/hackback/espionage-event/src/app/api/auth/verify-otp/route.ts#L50)), but it is never persisted to MongoDB, memory, or signed as a JWT. Downstream routes (`/api/round1/submit`, `/api/round2/execute`, `/api/dashboard/me`) do not validate any bearer token or cookie.
2. **Identity Spoofing in Contest Submissions (BOLA/IDOR)**: Endpoints `/api/round1/submit` ([src/app/api/round1/submit/route.ts:9](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round1/submit/route.ts#L9)) and `/api/round2/execute` ([src/app/api/round2/execute/route.ts:18](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round2/execute/route.ts#L18)) take `{ email }` directly from the unauthenticated JSON request payload. Any actor knowing another participant's email address can submit an empty test payload and prematurely lock that participant out.
3. **Client-Trusted Anti-Cheat Metrics**: `/api/round1/submit` directly assigns `participant.round1Warnings = warnings || 0` and `participant.round1KeyViolations = keyViolations || 0` ([src/app/api/round1/submit/route.ts:49-50](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round1/submit/route.ts#L49-L50)) from untrusted client input.
4. **Collision-Prone Participant ID Generation**: In `/api/register-manual/route.ts:58-64`, participant IDs are randomly chosen from a range of only 900 possible values (`ESP-100` to `ESP-999`). When a collision hits the MongoDB unique index, the database throws an unhandled duplicate key error (`E11000`), failing the registration with HTTP 500.
5. **Static Plaintext Shared Secret for Admin**: Admin endpoints throughout `src/app/api/admin/*` require `ADMIN_PASSWORD` in the JSON request body rather than using signed HTTP-only cookies or role-based sessions.

---

## Stage 8 — Verification & False Alarm Checks

- **Suspected Issue**: *Can a participant repeatedly submit Round 1 answers to overwrite their score or bypass the timer?*
  - **Verification**: Examined [src/app/api/round1/submit/route.ts:25-27](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round1/submit/route.ts#L25-L27). The route explicitly blocks repeated submissions:
    ```typescript
    if (participant.round1SubmittedAt) {
      return NextResponse.json({ error: 'Already submitted.' }, { status: 403 });
    }
    ```
    Once submitted, subsequent requests are rejected.
- **Suspected Issue**: *Can any user execute code in Round 2 without qualifying?*
  - **Verification**: Examined [src/app/api/round2/execute/route.ts:30](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round2/execute/route.ts#L30). The route explicitly enforces `if (!participant.isShortlisted) return NextResponse.json({ error: 'Not shortlisted.' }, { status: 403 });`.

---

## Final Observations (`OBSERVATIONS.md` Format)

### Observation 1: Session Token Generation Without Server Verification (Missing Auth State)
- **Title**: Unverified Ephemeral Session Tokens Across Protected Contest Endpoints
- **Status**: **CONFIRMED**
- **Exact File:Line**: [src/app/api/auth/verify-otp/route.ts:50-61](file:///Users/tanushrid/hackback/espionage-event/src/app/api/auth/verify-otp/route.ts#L50-L61) and [src/app/api/round1/submit/route.ts:9](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round1/submit/route.ts#L9)
- **What the Code Actually Does**: In `verify-otp`, `crypto.randomBytes(32).toString('hex')` generates an ephemeral token that is returned to the client in JSON. This token is never saved to the database, never signed as a JWT, and never stored in a session store. Downstream endpoints like `/api/round1/submit` accept raw `{ email }` and never inspect authorization headers, cookies, or session tokens.
- **Why It Matters**: The session token provides no security guarantee. Any client request specifying a target email address is treated as authentic without proof of session ownership.
- **Verification Reasoning**: Code inspection of `verify-otp/route.ts` confirms the token is instantiated as a local variable and returned directly in `NextResponse.json` without updating Mongoose models. Tracing `round1/submit/route.ts` and `round2/execute/route.ts` confirms that neither route reads an `Authorization` header or session cookie.

---

### Observation 2: Unauthenticated Contest Submission Allows Identity Substitution (BOLA/IDOR)
- **Title**: Identity Substitution via Unauthenticated Submission Endpoints
- **Status**: **CONFIRMED**
- **Exact File:Line**: [src/app/api/round1/submit/route.ts:9-27](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round1/submit/route.ts#L9-L27)
- **What the Code Actually Does**: `/api/round1/submit` parses `{ email, answers }` from `req.json()`. It searches for the participant document matching `email.toLowerCase()`, computes the score based on the answers provided, and records `participant.round1SubmittedAt = new Date()`.
- **Why It Matters**: Because authorization is missing, an adversary can submit an empty answer array `[]` on behalf of any known participant email. The server permanently marks that participant's record with `round1SubmittedAt = new Date()`, giving them a score of 0 and locking them out of Round 1 with an `'Already submitted.'` 403 error.
- **Verification Reasoning**: The endpoint checks only `config.round1Active` and `participant.round1SubmittedAt`. There is no check comparing caller identity to the requested resource identity.

---

### Observation 3: High-Collision Narrow Random Range for `participantId`
- **Title**: High Collision Probability on Participant ID Generation Causing Registration Failures
- **Status**: **CONFIRMED**
- **Exact File:Line**: [src/app/api/register-manual/route.ts:58-64](file:///Users/tanushrid/hackback/espionage-event/src/app/api/register-manual/route.ts#L58-L64)
- **What the Code Actually Does**: During self/manual registration, the server constructs the identifier using:
  ```typescript
  const rand = Math.floor(100 + Math.random() * 900);
  const participantId = `ESP-${rand}`;
  ```
  It then attempts `await Participant.create({ ..., participantId })`.
- **Why It Matters**: There are only 900 possible IDs (`ESP-100` through `ESP-999`). Under the Birthday Paradox, collisions begin occurring with high probability after only dozens of registrations. Because `participantId` has a `unique: true` constraint in [src/models/Participant.ts:40](file:///Users/tanushrid/hackback/espionage-event/src/models/Participant.ts#L40), collisions throw an unhandled MongoDB duplicate key error (`E11000`), terminating the request with HTTP 500.
- **Verification Reasoning**: Tracing `register-manual/route.ts` shows no retry loop, collision detection, or fallback sequence to select an alternative ID upon collision.

---

### Observation 4: Client-Controlled Anti-Cheat Violation Counters
- **Title**: Tamperable Anti-Cheat Metrics via Direct Client Trust
- **Status**: **CONFIRMED**
- **Exact File:Line**: [src/app/api/round1/submit/route.ts:49-50](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round1/submit/route.ts#L49-L50) and [src/app/api/round2/submit/route.ts:30-31](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round2/submit/route.ts#L30-L31)
- **What the Code Actually Does**: The server sets violation counts directly from client parameters:
  ```typescript
  participant.round1Warnings = warnings || 0;
  participant.round1KeyViolations = keyViolations || 0;
  ```
- **Why It Matters**: Although client-side UI scripts monitor fullscreen changes and tab switches, any participant can modify the HTTP request body in browser devtools to submit `warnings: 0` and `keyViolations: 0`, completely erasing recorded infractions before they reach the database.
- **Verification Reasoning**: Confirmed by examining lines 9 and 49-50 of `src/app/api/round1/submit/route.ts`. No server-signed tokens, HMAC signatures, or incremental server-side log counters exist to validate client-reported violation metrics.

---

### Observation 5: Suspected Score Overwrite and Re-submission Vulnerability in Round 1
- **Title**: [FALSE ALARM CORRECTION] Round 1 Re-submission and Score Overwrite
- **Status**: **NOT CONFIRMED (False Alarm)**
- **Exact File:Line**: [src/app/api/round1/submit/route.ts:25-27](file:///Users/tanushrid/hackback/espionage-event/src/app/api/round1/submit/route.ts#L25-L27)
- **What the Code Actually Does**: Checks if `participant.round1SubmittedAt` is already set:
  ```typescript
  if (participant.round1SubmittedAt) {
    return NextResponse.json({ error: 'Already submitted.' }, { status: 403 });
  }
  ```
- **Why It Matters / Correction**: An initial assessment of the route suggested that because scoring is computed dynamically from submitted answers, a participant might submit iteratively to maximize points. However, verification reveals that the server updates `participant.round1SubmittedAt = new Date()` on line 48 and saves it on line 51. Any subsequent submission attempt triggers the guard on line 25, returning HTTP 403. Multiple submissions or score overwrites are impossible.
