# PROJECT_DESIGN.md

## 1. Project Overview

This project is a browser-based, real-time word classification game where players tap floating word bubbles that match a currently active title prompt (for example, **“Tap the animals”**).

Core gameplay loop:
- A timed session runs for **120 seconds**.
- The game continuously spawns floating bubbles containing words.
- The active prompt/title changes every **15–30 seconds** (randomized interval).
- The player gains/loses points by clicking bubbles based on whether each clicked word matches the current title’s conceptual criteria.

High-level architecture:
- **Frontend**: React single-page app (SPA), HTML/CSS-rendered bubbles, JavaScript physics loop (`requestAnimationFrame`) for movement and collisions.
- **Backend**: Node.js + Express API for session lifecycle, round/title generation, and authoritative click validation/scoring.
- **Database**: PostgreSQL (Neon in production).
- **Deployment**: Vercel for frontend + server API routes/functions, Neon for PostgreSQL.

Design principles:
- Backend is authoritative for game correctness and score updates.
- Frontend owns animation/physics and rendering performance.
- Data model supports many-to-many concept mapping so words can satisfy multiple conceptual prompts.
- The system supports local development and cloud deployment with environment-based configuration.

---

## 2. Game Mechanics

### 2.1 Session Lifecycle

1. Player selects:
   - Word Difficulty: `easy | medium | hard`
   - Speed Difficulty: `normal | fast | extreme | usain_bolt`
2. Frontend calls `POST /api/game/start`.
3. Backend creates a `game_session`, stores selected difficulties, computes `started_at` and `ends_at = started_at + 120s`, and returns first title/round metadata.
4. Frontend starts:
   - Global 120-second countdown
   - Bubble spawn scheduler
   - Physics loop
5. Every 15–30 seconds, frontend requests a new round via `POST /api/game/next-round`.
6. Player clicks bubbles; each click triggers `POST /api/game/click` for authoritative validation and score update.
7. At 120 seconds (or when backend reports session ended), frontend shows game-over summary.

### 2.2 Scoring Rules

- Correct click: **+1**
- Incorrect click: **-1**
- Bubble leaving top boundary without click: **0**

Scoring source of truth:
- The backend computes and persists score changes in `game_sessions.score`.
- Frontend displays score from backend response to avoid divergence.

### 2.3 Prompt/Title Change Behavior

When a title changes:
- Bubbles currently on screen are evaluated by frontend against new round correctness set (`correct_word_ids` supplied by backend for that round).
- Any bubble whose `wordId` is **not valid** for the new title is removed immediately.
- Valid bubbles remain in play and keep their current trajectory.

This ensures semantic consistency after each title change while preserving motion continuity for relevant bubbles.

---

## 3. Difficulty Systems

Two independent difficulty axes combine to create final game behavior.

### 3.1 Word Difficulty (content complexity)

Word records have numeric `difficulty`:
- `1` = easy
- `2` = medium
- `3` = hard

Mode rules:
- **easy**: include only words with difficulty `1`
- **medium**: include words with difficulty `1–2`
- **hard**: include words with difficulty `2–3`

Impact:
- Affects candidate pools for both correct and incorrect bubbles.
- Ensures semantic challenge progression independent of speed/physics challenge.

### 3.2 Speed Difficulty (kinetic challenge)

Speed difficulty controls:
- Bubble spawn rate
- Upward speed (`vy` magnitude)
- Horizontal drift (`vx` range)

Recommended baseline presets (tunable constants):

| Mode | Spawn Interval (ms) | Upward Speed `vy` (px/s) | Horizontal Drift `vx` (px/s) |
|---|---:|---:|---:|
| normal | 1400–1800 | -40 to -70 | -20 to 20 |
| fast | 900–1300 | -70 to -110 | -35 to 35 |
| extreme | 550–900 | -110 to -160 | -50 to 50 |
| usain_bolt | 300–600 | -160 to -230 | -70 to 70 |

Notes:
- Negative `vy` means upward motion in typical browser coordinate space.
- Spawn intervals should include jitter to avoid mechanical patterns.
- Collision response should preserve overall intensity at higher speed tiers.

### 3.3 Difficulty Combination

The effective game challenge is the Cartesian product:
- `(word difficulty)` × `(speed difficulty)`.

Examples:
- `easy + usain_bolt`: simple vocabulary, very high reaction demand.
- `hard + normal`: complex categorization, slower motor demand.

---

## 4. Bubble Physics System

The frontend implements a lightweight real-time physics loop with `requestAnimationFrame`.

### 4.1 Bubble Data Model (client runtime)

Each bubble object contains:
- `bubbleId` (unique client instance id)
- `wordId` (database id)
- `word` (display text)
- `x`, `y` (center position)
- `vx`, `vy` (velocity components)
- `radius`
- `spawnTime`
- `clicked` (boolean)

### 4.2 Spawn Rules

At spawn:
- Sample candidate word from current round pool (mix of correct + incorrect).
- Determine radius based on text length + min/max constraints.
- Spawn near bottom region with randomized x-position.
- Enforce **non-overlap at spawn**:
  - Attempt random positions up to `N` retries.
  - Accept only if distance to every existing bubble center > sum of radii + padding.
  - If retries exhausted, skip spawn event (or reduce radius in fallback policy).

### 4.3 Frame Update Loop

Per frame (`deltaTime` based):
1. Update position: `x += vx * dt`, `y += vy * dt`.
2. Handle boundary collisions:
   - Left/right walls: reflect `vx` and clamp position.
   - Top boundary: remove bubble (no score).
   - Bottom boundary: optional clamp/reflect depending on spawn strategy (typically not needed if spawn below active region and movement is upward).
3. Resolve bubble-bubble collisions:
   - Broad phase: simple pairwise checks (acceptable for small/moderate bubble counts).
   - Narrow phase: circle intersection test.
   - Response: separate overlapping circles and apply elastic-ish velocity exchange along collision normal.
4. Remove clicked bubbles.
5. Render updated state.

### 4.4 Collision Model

Recommended simplified elastic collision:
- Treat bubbles as equal mass circles.
- On overlap:
  - Compute normal vector `n = normalize(p2 - p1)`.
  - Push circles apart by overlap/2 each.
  - Project relative velocity on `n`; if moving toward each other, swap/adjust normal components.

This yields believable bouncing behavior with low computational complexity.

### 4.5 Performance Constraints

- Target 60 FPS where possible.
- Keep active bubble count bounded per speed mode.
- Use `useRef` and batched React state updates to minimize rerenders.
- Consider rendering bubbles in a single positioned layer for efficient DOM updates.

---

## 5. Database Schema

PostgreSQL relational schema enabling many-to-many mappings.

### 5.1 Tables

#### `words`
- `id` (PK, serial/bigserial or UUID)
- `word` (text, unique, not null)
- `difficulty` (smallint/check in 1..3, not null)

#### `concepts`
- `id` (PK)
- `name` (text, unique, not null)

#### `concept_words`
- `concept_id` (FK -> concepts.id, not null)
- `word_id` (FK -> words.id, not null)
- Composite PK: (`concept_id`, `word_id`)

#### `titles`
- `id` (PK)
- `title` (text, unique, not null)  
  Example: “Tap the animals”

#### `title_concepts`
- `title_id` (FK -> titles.id, not null)
- `concept_id` (FK -> concepts.id, not null)
- Composite PK: (`title_id`, `concept_id`)

#### `game_sessions`
- `id` (PK)
- `score` (integer, default 0, not null)
- `word_difficulty` (text enum-ish: easy/medium/hard, not null)
- `speed_difficulty` (text enum-ish: normal/fast/extreme/usain_bolt, not null)
- `started_at` (timestamptz, not null)
- `ends_at` (timestamptz, not null)
- `status` (text enum-ish: active/completed/abandoned, not null)

### 5.2 Relationship Semantics

This model supports multiple conceptual memberships naturally.

Example mapping:
- `banana` in `words`
- `fruit` in `concepts`
- `yellow` in `concepts`
- `concept_words` rows:
  - (`fruit`, `banana`)
  - (`yellow`, `banana`)

Therefore, the same word can be valid for different titles depending on title-concept mapping.

Title linkage:
- `titles.title = "Tap the fruits"` links via `title_concepts` to concept `fruit`.
- `titles.title = "Tap the yellow things"` links via `title_concepts` to concept `yellow`.

A title can map to one or many concepts (e.g., composite prompts in future expansion).

### 5.3 Suggested Constraints & Indexes

Constraints:
- Check constraint on `words.difficulty` in `(1,2,3)`.
- Check constraint on `game_sessions.status` allowed values.
- Check constraint on difficulty text fields.

Indexes:
- `words(difficulty)`
- `concept_words(word_id)` and PK index
- `title_concepts(title_id)` and PK index
- `game_sessions(status, ends_at)` for cleanup/queries

---

## 6. Puzzle Generation Algorithm

(Round generation for each prompt cycle)

### 6.1 Inputs

- `session_id`
- `word_difficulty` from session
- Optional anti-repeat constraints (recent titles/words)

### 6.2 Steps

1. **Choose random title**
   - Select one title from `titles`.
   - Optional: avoid immediate repeat of current/last title.

2. **Resolve title concepts**
   - Query `title_concepts` for selected `title_id`.

3. **Fetch all correct candidate words**
   - Join `concept_words` + `words` where concept is in selected title concepts.
   - Distinct by word id.

4. **Apply word difficulty filter**
   - Convert session mode to allowed difficulty set:
     - easy -> `{1}`
     - medium -> `{1,2}`
     - hard -> `{2,3}`
   - Keep only matching words.

5. **Select subset for correct pool**
   - Randomly choose `N_correct` (configurable by speed mode and UI density).
   - Ensure enough variety and no duplicates.

6. **Select incorrect pool**
   - Query words that **do not belong** to any selected title concept.
   - Apply same difficulty filter.
   - Randomly select `N_incorrect`.

7. **Return round payload**
   - `round_id` (logical round token)
   - `title_id`, `title`
   - `correct_word_ids` (authoritative correctness set)
   - `word_pool` (display candidates containing both correct and incorrect words)
   - `title_expires_at` (optional, for synchronized transitions)

### 6.3 Correctness Logic

On click validation:
- A click is correct if `clicked_word_id ∈ correct_word_ids` for the current active round of the session.

Server authoritative behavior:
- Backend maintains session’s current round context (or validates via signed round token payload).
- Frontend cannot self-authorize correctness.

### 6.4 Edge Cases

- If insufficient correct words after filtering:
  - Fallback to smaller `N_correct`.
  - Optionally reselect title.
- If insufficient incorrect words:
  - Reduce total spawn diversity temporarily.
- Prevent duplicate words flooding the same short interval unless intended.

---

## 7. API Design

Base path: `/api`

### 7.1 `GET /api/health`

Purpose:
- Health check for runtime and optional DB connectivity.

Response (200):
```json
{
  "status": "ok",
  "service": "wubble-web-api",
  "time": "2025-01-01T00:00:00.000Z"
}
```

### 7.2 `POST /api/game/start`

Purpose:
- Start a new 120-second session.

Request body:
```json
{
  "wordDifficulty": "easy",
  "speedDifficulty": "normal"
}
```

Server actions:
1. Validate difficulty enums.
2. Create `game_sessions` row with score `0`, status `active`, timestamps.
3. Generate initial round/title using puzzle algorithm.
4. Store session round context (in DB table or in-memory/cache keyed by session id in serverless-friendly way, preferably DB-backed).

Response (201):
```json
{
  "sessionId": "...",
  "score": 0,
  "startedAt": "...",
  "endsAt": "...",
  "wordDifficulty": "easy",
  "speedDifficulty": "normal",
  "round": {
    "roundId": "...",
    "titleId": 12,
    "title": "Tap the animals",
    "correctWordIds": [3, 9, 14],
    "wordPool": [
      {"id": 3, "word": "dog"},
      {"id": 9, "word": "lion"},
      {"id": 22, "word": "banana"}
    ],
    "nextTitleWindowSec": {"min": 15, "max": 30}
  }
}
```

### 7.3 `POST /api/game/next-round`

Purpose:
- Generate and return the next title/round for an existing active session.

Request body:
```json
{
  "sessionId": "..."
}
```

Server actions:
1. Validate session exists and is active and not expired.
2. Generate new round using same session `wordDifficulty`.
3. Persist/update current round context.

Response (200):
```json
{
  "sessionId": "...",
  "round": {
    "roundId": "...",
    "titleId": 15,
    "title": "Tap the fruits",
    "correctWordIds": [22, 24, 27],
    "wordPool": [
      {"id": 22, "word": "banana"},
      {"id": 24, "word": "apple"},
      {"id": 31, "word": "truck"}
    ]
  }
}
```

### 7.4 `POST /api/game/click`

Purpose:
- Validate a bubble click and update score authoritatively.

Request body:
```json
{
  "sessionId": "...",
  "roundId": "...",
  "wordId": 22,
  "clientTime": "2025-01-01T00:00:00.000Z"
}
```

Server actions:
1. Validate active session and round ownership.
2. Determine correctness using authoritative `correct_word_ids` for round.
3. Apply score delta:
   - correct => `+1`
   - incorrect => `-1`
4. Update `game_sessions.score` atomically.
5. Return current score and result.

Response (200):
```json
{
  "sessionId": "...",
  "roundId": "...",
  "wordId": 22,
  "correct": true,
  "scoreDelta": 1,
  "score": 17,
  "sessionStatus": "active"
}
```

### 7.5 Error Handling (all endpoints)

Standard error shape:
```json
{
  "error": {
    "code": "SESSION_EXPIRED",
    "message": "Game session has ended"
  }
}
```

Suggested error codes:
- `VALIDATION_ERROR`
- `SESSION_NOT_FOUND`
- `SESSION_EXPIRED`
- `ROUND_NOT_FOUND`
- `ROUND_SESSION_MISMATCH`
- `INTERNAL_ERROR`

---

## 8. Frontend Architecture

React component hierarchy:

- `App`
  - `GameMenu`
  - `GameScreen`
    - `HeaderBar`
    - `PromptDisplay`
    - `BubbleField`
      - `Bubble` (many)
  - `GameOverModal`

### 8.1 Component Responsibilities

#### `App`
- Global screen state: menu vs active game vs game over.
- Holds session-level data (sessionId, score, timers).
- Coordinates API service integration.

#### `GameMenu`
- Difficulty selectors (`wordDifficulty`, `speedDifficulty`).
- Start button triggering `POST /api/game/start`.

#### `GameScreen`
- Main play container for active session.
- Orchestrates title change timer (15–30 sec randomized).
- Triggers `next-round` calls.

#### `HeaderBar`
- Displays score, remaining time, active speed mode.

#### `PromptDisplay`
- Displays current prompt/title prominently.
- Optional transition animation on title changes.

#### `BubbleField`
- Owns bubble state array and physics loop.
- Spawns bubbles per speed configuration.
- Handles collisions, boundaries, and removals.
- On click, calls parent callback to invoke `/api/game/click`.
- On round change, removes bubbles not valid under new `correctWordIds`.

#### `Bubble`
- Pure presentational component for one bubble.
- Receives position, size, text, click handler.

#### `GameOverModal`
- Displays final score and replay option.
- Replay returns to menu or restarts with same settings.

### 8.2 Frontend State Model (suggested)

Session state:
- `sessionId`
- `score`
- `wordDifficulty`
- `speedDifficulty`
- `startedAt`
- `endsAt`
- `sessionStatus`

Round state:
- `roundId`
- `titleId`
- `title`
- `correctWordIds` (set for fast membership checks)
- `wordPool`

Runtime bubble state:
- `bubbles[]` with physics properties
- spawn scheduler refs
- frame timestamp refs

### 8.3 Networking Layer

- `apiClient` module wrapping fetch calls.
- Handles JSON parsing, typed responses, and centralized error mapping.
- Includes retry policy only for safe operations (`health`, optional `next-round`), not for `click` unless idempotency token is introduced.

---

## 9. Project Directory Structure

Recommended monorepo-like structure:

```text
Wubble-Web/
├─ frontend/
│  ├─ src/
│  │  ├─ components/
│  │  │  ├─ App/
│  │  │  ├─ GameMenu/
│  │  │  ├─ GameScreen/
│  │  │  ├─ HeaderBar/
│  │  │  ├─ PromptDisplay/
│  │  │  ├─ BubbleField/
│  │  │  ├─ Bubble/
│  │  │  └─ GameOverModal/
│  │  ├─ hooks/
│  │  │  ├─ useGameTimer.js
│  │  │  └─ useBubblePhysics.js
│  │  ├─ services/
│  │  │  └─ apiClient.js
│  │  ├─ constants/
│  │  │  └─ difficultyConfig.js
│  │  ├─ styles/
│  │  └─ main.jsx
│  └─ package.json
├─ server/
│  ├─ src/
│  │  ├─ app.js
│  │  ├─ routes/
│  │  │  ├─ health.js
│  │  │  └─ game.js
│  │  ├─ controllers/
│  │  ├─ services/
│  │  │  ├─ sessionService.js
│  │  │  ├─ roundService.js
│  │  │  └─ scoringService.js
│  │  ├─ db/
│  │  │  ├─ client.js
│  │  │  └─ queries/
│  │  ├─ middleware/
│  │  └─ utils/
│  └─ package.json
├─ db/
│  ├─ migrations/
│  ├─ seeds/
│  └─ schema.sql
├─ docs/
│  ├─ api-spec.md
│  ├─ data-dictionary.md
│  └─ gameplay-balancing.md
└─ PROJECT_DESIGN.md
```

Directory purpose:
- `frontend/`: all client UI, physics, and game interaction logic.
- `server/`: Express API and game/session business rules.
- `db/`: schema, migration scripts, seed data for words/concepts/titles.
- `docs/`: supplementary docs derived from this master design.
- `PROJECT_DESIGN.md`: authoritative architecture specification.

---

## 10. Deployment Strategy

### 10.1 Environment Variables

Required:
- `DATABASE_URL` (PostgreSQL connection string; Neon in production)

Optional recommended:
- `NODE_ENV`
- `CORS_ORIGIN`
- `LOG_LEVEL`

Use `.env` locally and Vercel project environment settings in production.

### 10.2 Local Development Model

- Run Postgres locally (Docker or local instance) or connect to Neon dev branch.
- Set `DATABASE_URL` in local env.
- Start backend (`server/`) on local port (e.g., 3001).
- Start frontend (`frontend/`) on local port (e.g., 5173/3000).
- Configure frontend API base URL for local backend.

Typical local flow:
1. Apply migrations from `db/migrations`.
2. Seed words/concepts/titles.
3. Start server and frontend concurrently.
4. Verify `/api/health` and full game loop.

### 10.3 Vercel + Neon Production Model

Frontend:
- React app built into static assets and hosted by Vercel.

Backend:
- Express handlers adapted as Vercel serverless functions (or route handlers) under `/api/*`.
- Stateless compute: session/round authoritative data should be persisted (DB-backed), not in process memory.

Database:
- Neon Postgres via `DATABASE_URL`.
- Use connection pooling compatible with serverless environments.

### 10.4 Runtime Considerations

- Serverless cold starts: keep handlers lightweight.
- DB query efficiency: indexed joins for round generation.
- Session expiration checks on every game endpoint.
- Observability:
  - health endpoint
  - structured logs for round generation/click validation

### 10.5 Release & Migration Strategy

- CI pipeline should run:
  - lint/tests (frontend/server)
  - migration checks
- Deploy sequence:
  1. Apply DB migrations
  2. Deploy backend/frontend
  3. Validate with `/api/health`
- Backward compatibility:
  - Avoid breaking API payloads once frontend released.

---

## Appendix: Implementation Notes for Future Tasks

- Keep backend authoritative for correctness and score to prevent cheating.
- Ensure `next-round` responses include `correctWordIds` so frontend can purge invalid on-screen bubbles immediately.
- Maintain deterministic difficulty mapping in shared constants to reduce mismatch between frontend UI and backend filtering.
- Treat this document as the baseline contract; if implementation diverges, update this file first and then code.
