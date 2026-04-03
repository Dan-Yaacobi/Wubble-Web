# Wubble Web

Monorepo scaffold for the Wubble Web real-time word bubble game.

## Project Layout

- `frontend/` — React + Vite frontend app scaffold
- `server/` — Node + Express API scaffold
- `db/` — database schema, migrations, and seed directories
- `docs/` — supporting documentation

## Run Frontend

1. Install dependencies:
   ```bash
   cd frontend
   npm install
   ```
2. Start dev server:
   ```bash
   npm run dev
   ```

## Run Server

1. Install dependencies:
   ```bash
   cd server
   npm install
   ```
2. Copy environment file and set values:
   ```bash
   cp .env.example .env
   ```
3. Ensure `DATABASE_URL` is configured in `.env`.
4. Start server:
   ```bash
   npm run dev
   ```

Health check endpoint:

- `GET /api/health` → `{ "status": "ok" }`
