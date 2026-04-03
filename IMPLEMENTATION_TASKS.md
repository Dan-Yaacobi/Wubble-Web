# IMPLEMENTATION_TASKS.md

This roadmap translates `PROJECT_DESIGN.md` into sequential, implementation-ready tasks without changing the defined architecture.

## Phase 1 — Project Setup

### Task 1: Initialize Repository Structure

**Goal:**  
Create the top-level project directories and baseline files for frontend, server, database, and docs according to the specified structure.

**Files Affected:**  
`frontend/`  
`server/`  
`db/`  
`docs/`  
`.gitignore`  
`README.md` (update setup sections)

**Dependencies:**  
None

**Expected Output:**  
A scaffolded repository matching the architecture layout in `PROJECT_DESIGN.md`, ready for package initialization in each workspace.

---

### Task 2: Configure Frontend and Server Package Baselines

**Goal:**  
Initialize Node package manifests and scripts for both `frontend` and `server`, including dev/build/start/lint placeholders.

**Files Affected:**  
`frontend/package.json`  
`server/package.json`  
`frontend/.env.example`  
`server/.env.example`

**Dependencies:**  
Task 1

**Expected Output:**  
Both app layers have runnable package definitions and environment variable templates aligned to local and Vercel workflows.

---

### Task 3: Establish Shared Configuration and Difficulty Constants

**Goal:**  
Define canonical difficulty mappings (word + speed presets) so frontend and backend use consistent behavior.

**Files Affected:**  
`frontend/src/constants/difficultyConfig.js`  
`server/src/utils/difficultyConfig.js`  
`docs/gameplay-balancing.md`

**Dependencies:**  
Task 2

**Expected Output:**  
Documented and implemented constants for `easy|medium|hard` and `normal|fast|extreme|usain_bolt`, including spawn and velocity ranges.

---

## Phase 2 — Database Layer

### Task 4: Implement Base PostgreSQL Schema

**Goal:**  
Create the core relational schema for words, concepts, title mappings, sessions, and constraints/indexes.

**Files Affected:**  
`db/schema.sql`  
`db/migrations/001_initial_schema.sql`

**Dependencies:**  
Task 1

**Expected Output:**  
A migration-backed schema implementing all required tables, PK/FK relationships, checks, and indexes from the design.

---

### Task 5: Add Round Context Persistence Schema

**Goal:**  
Create DB structures for authoritative round tracking per session in a serverless-safe, DB-backed manner.

**Files Affected:**  
`db/migrations/002_round_context.sql`  
`db/schema.sql` (append new objects)  
`docs/data-dictionary.md`

**Dependencies:**  
Task 4

**Expected Output:**  
Database tables (or equivalent DB-backed model) exist to store active/current round metadata, including `round_id` and correctness sets linked to sessions.

---

### Task 6: Create Seed Data for Words, Concepts, and Titles

**Goal:**  
Populate development data that supports multiple concepts per word and multiple concepts per title.

**Files Affected:**  
`db/seeds/001_words.sql`  
`db/seeds/002_concepts.sql`  
`db/seeds/003_concept_words.sql`  
`db/seeds/004_titles.sql`  
`db/seeds/005_title_concepts.sql`

**Dependencies:**  
Task 4

**Expected Output:**  
Seed scripts provide representative content across all difficulty levels and conceptual mappings for gameplay/testing.

---

### Task 7: Add DB Tooling Scripts for Migrate/Seed/Reset

**Goal:**  
Provide repeatable local commands to apply migrations and seed data quickly.

**Files Affected:**  
`db/README.md`  
`package.json` (root or workspace scripts)  
`server/package.json` (DB script passthrough if needed)

**Dependencies:**  
Tasks 4, 6

**Expected Output:**  
Developers can run scripted commands to initialize and refresh the database for local development.

---

## Phase 3 — Backend Core Systems

### Task 8: Implement Server Bootstrap, Middleware, and Health Route

**Goal:**  
Set up Express app foundation with JSON handling, CORS config, error middleware, and `/api/health` endpoint.

**Files Affected:**  
`server/src/app.js`  
`server/src/routes/health.js`  
`server/src/middleware/errorHandler.js`  
`server/src/utils/httpErrors.js`

**Dependencies:**  
Task 2

**Expected Output:**  
Server starts reliably and returns standardized health payload and error responses.

---

### Task 9: Implement Database Client and Query Modules

**Goal:**  
Create reusable DB access layer (pooled client + query helpers) compatible with local and Neon deployments.

**Files Affected:**  
`server/src/db/client.js`  
`server/src/db/queries/sessionQueries.js`  
`server/src/db/queries/roundQueries.js`  
`server/src/db/queries/wordQueries.js`  
`server/src/db/queries/titleQueries.js`

**Dependencies:**  
Tasks 4, 5, 8

**Expected Output:**  
Backend services can call typed/reusable query functions rather than embedding raw SQL in controllers.

---

### Task 10: Build Session Lifecycle Service

**Goal:**  
Implement creation and validation logic for 120-second sessions, including active/expired status handling.

**Files Affected:**  
`server/src/services/sessionService.js`  
`server/src/utils/time.js`

**Dependencies:**  
Task 9

**Expected Output:**  
Service functions exist to create sessions, fetch sessions, and enforce expiration/status checks on API calls.

---

### Task 11: Build Round Generation Service (Puzzle Algorithm)

**Goal:**  
Implement title selection, concept resolution, difficulty filtering, and correct/incorrect word pool generation.

**Files Affected:**  
`server/src/services/roundService.js`  
`server/src/db/queries/roundQueries.js` (extend)

**Dependencies:**  
Tasks 3, 9, 10

**Expected Output:**  
Round generator returns payloads matching design (`roundId`, `title`, `correctWordIds`, `wordPool`) with edge-case handling for low pool sizes.

---

### Task 12: Build Scoring Service with Atomic Updates

**Goal:**  
Implement authoritative click validation and transactional score updates based on current round context.

**Files Affected:**  
`server/src/services/scoringService.js`  
`server/src/db/queries/sessionQueries.js` (atomic update query)

**Dependencies:**  
Tasks 5, 9, 10, 11

**Expected Output:**  
Backend can validate `wordId` membership in authoritative round data and persist score deltas (+1/-1) safely.

---

## Phase 4 — API Endpoints

### Task 13: Implement Game Start Endpoint

**Goal:**  
Add `POST /api/game/start` to validate inputs, create session, generate initial round, and return required payload.

**Files Affected:**  
`server/src/routes/game.js`  
`server/src/controllers/gameController.js`  
`server/src/services/sessionService.js` (integration)

**Dependencies:**  
Tasks 10, 11

**Expected Output:**  
Clients can start a new game session and receive initial round data with timing and difficulty context.

---

### Task 14: Implement Next Round Endpoint

**Goal:**  
Add `POST /api/game/next-round` with active-session validation and round-context persistence.

**Files Affected:**  
`server/src/routes/game.js`  
`server/src/controllers/gameController.js`  
`server/src/services/roundService.js` (integration)

**Dependencies:**  
Tasks 5, 10, 11, 13

**Expected Output:**  
Active sessions can request new rounds every 15–30 seconds and receive updated prompt/correctness payloads.

---

### Task 15: Implement Click Validation Endpoint

**Goal:**  
Add `POST /api/game/click` to validate round ownership, compute correctness, update score, and return result.

**Files Affected:**  
`server/src/routes/game.js`  
`server/src/controllers/gameController.js`  
`server/src/services/scoringService.js` (integration)

**Dependencies:**  
Tasks 12, 13, 14

**Expected Output:**  
Frontend receives authoritative click results (`correct`, `scoreDelta`, `score`, `sessionStatus`) per click.

---

### Task 16: Standardize API Validation, Error Codes, and Spec Docs

**Goal:**  
Apply consistent request validation and error response format across all endpoints and document contracts.

**Files Affected:**  
`server/src/middleware/validateRequest.js`  
`server/src/utils/errorCodes.js`  
`docs/api-spec.md`

**Dependencies:**  
Tasks 8, 13, 14, 15

**Expected Output:**  
All API routes return standardized error objects and the docs reflect exact request/response schemas.

---

## Phase 5 — Frontend Architecture

### Task 17: Scaffold Core React App Screens and Navigation Flow

**Goal:**  
Implement `App`, `GameMenu`, `GameScreen`, and `GameOverModal` screen-state flow without physics logic yet.

**Files Affected:**  
`frontend/src/main.jsx`  
`frontend/src/components/App/App.jsx`  
`frontend/src/components/GameMenu/GameMenu.jsx`  
`frontend/src/components/GameScreen/GameScreen.jsx`  
`frontend/src/components/GameOverModal/GameOverModal.jsx`

**Dependencies:**  
Task 2

**Expected Output:**  
User can move through menu → active game shell → game-over screen via component state transitions.

---

### Task 18: Implement Frontend API Client and Session/Round State Wiring

**Goal:**  
Create API client wrappers and connect start/next-round/click requests to app-level state.

**Files Affected:**  
`frontend/src/services/apiClient.js`  
`frontend/src/components/App/App.jsx`  
`frontend/src/components/GameScreen/GameScreen.jsx`

**Dependencies:**  
Tasks 13, 14, 15, 17

**Expected Output:**  
Frontend can start sessions, fetch next rounds, send clicks, and reflect backend-authoritative score/title state.

---

### Task 19: Implement Header, Prompt, and Countdown Timer Hook

**Goal:**  
Add HUD components and timer logic tied to `endsAt` for reliable 120-second session countdown.

**Files Affected:**  
`frontend/src/components/HeaderBar/HeaderBar.jsx`  
`frontend/src/components/PromptDisplay/PromptDisplay.jsx`  
`frontend/src/hooks/useGameTimer.js`  
`frontend/src/components/GameScreen/GameScreen.jsx`

**Dependencies:**  
Tasks 17, 18

**Expected Output:**  
Game screen displays score, remaining time, speed mode, and current prompt; game transitions at session end.

---

## Phase 6 — Game Mechanics

### Task 20: Build Bubble Rendering and Spawn Scheduler

**Goal:**  
Implement `BubbleField` and `Bubble` with speed-tier-driven spawn timing and candidate selection from round word pool.

**Files Affected:**  
`frontend/src/components/BubbleField/BubbleField.jsx`  
`frontend/src/components/Bubble/Bubble.jsx`  
`frontend/src/components/GameScreen/GameScreen.jsx`

**Dependencies:**  
Tasks 3, 18, 19

**Expected Output:**  
Bubbles spawn continuously with randomized positions/velocities based on selected speed difficulty.

---

### Task 21: Implement Physics Loop, Boundaries, and Bubble-Bubble Collision

**Goal:**  
Add `requestAnimationFrame` loop with delta-time updates, wall handling, top removal, overlap resolution, and elastic-ish collisions.

**Files Affected:**  
`frontend/src/hooks/useBubblePhysics.js`  
`frontend/src/components/BubbleField/BubbleField.jsx`

**Dependencies:**  
Task 20

**Expected Output:**  
Bubble motion and collisions run smoothly with bounded bubble counts and stable on-screen behavior.

---

### Task 22: Implement Click Handling and Round-Change Bubble Purge

**Goal:**  
Wire bubble clicks to backend scoring and remove invalid bubbles immediately when title changes.

**Files Affected:**  
`frontend/src/components/BubbleField/BubbleField.jsx`  
`frontend/src/components/GameScreen/GameScreen.jsx`  
`frontend/src/services/apiClient.js`

**Dependencies:**  
Tasks 15, 20, 21

**Expected Output:**  
Clicks trigger authoritative score updates, and round transitions retain only bubbles whose `wordId` remains correct under new `correctWordIds`.

---

## Phase 7 — UI Polish and Reliability

### Task 23: Add Visual Styling, Prompt Transitions, and Feedback States

**Goal:**  
Implement production-ready styling and UX feedback for gameplay status, loading, and error states.

**Files Affected:**  
`frontend/src/styles/`  
`frontend/src/components/PromptDisplay/PromptDisplay.jsx` (transition hooks/classes)  
`frontend/src/components/HeaderBar/HeaderBar.jsx`  
`frontend/src/components/GameOverModal/GameOverModal.jsx`

**Dependencies:**  
Tasks 19, 22

**Expected Output:**  
Game UI is polished with clear visual hierarchy, prompt transition cues, and user-visible network/error feedback.

---

### Task 24: Add Backend and Frontend Test Coverage for Core Flows

**Goal:**  
Create focused tests for session lifecycle, round generation, scoring correctness, and key frontend state transitions.

**Files Affected:**  
`server/tests/`  
`frontend/src/**/*.test.*`  
`server/package.json` (test scripts)  
`frontend/package.json` (test scripts)

**Dependencies:**  
Tasks 13, 14, 15, 18, 22

**Expected Output:**  
Automated tests validate core game correctness and reduce regressions across API and UI behavior.

---

## Phase 8 — Deployment and Operations

### Task 25: Configure Local + Vercel Deployment Pipeline

**Goal:**  
Finalize environment configuration, Vercel deployment setup, and runbook steps for migration + release.

**Files Affected:**  
`vercel.json`  
`README.md` (local + production setup)  
`docs/deployment.md`  
`server/src/app.js` (serverless entry compatibility if needed)

**Dependencies:**  
Tasks 7, 8, 16, 24

**Expected Output:**  
Project can be run locally with documented commands and deployed to Vercel/Neon with clear migration and health-check verification steps.
