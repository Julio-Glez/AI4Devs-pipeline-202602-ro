# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

LTI (Talent Tracking System) — a full-stack recruitment management system for managing candidates, job positions, and interview workflows. It is a monorepo with a React frontend (port 3000) and Express/Node.js backend (port 3010), backed by PostgreSQL via Docker.

## Development Setup

Prerequisites: Docker must be running for the database.

```bash
# 1. Start PostgreSQL
docker-compose up -d

# 2. Apply migrations and generate Prisma client
cd backend
npx prisma migrate dev
npx prisma generate

# 3. Seed the database
npx ts-node prisma/seed.ts

# 4. Start backend (from backend/)
npm run dev

# 5. Start frontend (from frontend/, separate terminal)
npm start
```

## Common Commands

### Backend (`backend/`)
```bash
npm run dev          # Dev server with live reload (ts-node-dev)
npm run build        # Compile TypeScript → dist/
npm start            # Run compiled server
npm test             # Run Jest tests
npm test -- --testPathPattern=<file>  # Run a single test file
```

### Frontend (`frontend/`)
```bash
npm start            # React dev server
npm run build        # Production build
npm test             # Jest tests
npm run cypress:open # Cypress E2E UI
npm run cypress:run  # Cypress E2E headless
```

### Database
```bash
npx prisma migrate dev       # Apply migrations
npx prisma generate          # Regenerate Prisma client after schema changes
npx prisma studio            # Visual DB browser
```

## Architecture

The project follows **Clean Architecture with Domain-Driven Design (DDD)**:

```
backend/src/
├── domain/models/          # Business entities (Candidate, Position, Interview, etc.)
├── application/services/   # Use cases & business logic (candidateService, positionService)
├── presentation/controllers/ # HTTP request handlers
├── infrastructure/         # Prisma ORM wrappers
└── routes/                 # Express route definitions
```

**Request flow**: Route → Controller → Service → Domain Model → Prisma

The frontend uses React with react-router-dom v6 for routing. Components in `frontend/src/components/` call API services defined in `frontend/src/services/`.

### Key Domain Models (Prisma)

Candidate → Application → Interview tracks the candidate's journey through a Position's InterviewFlow → InterviewSteps.

### API

- `POST/GET /candidates`, `PUT /candidates/:id` — candidate management; stage updates
- `GET /positions/:id` — position details with interview flow
- `POST /upload` — CV file upload (Multer, stored in `uploads/`)
- Swagger docs available at runtime; full spec in `backend/api-spec.yaml`

### CI/CD

`.github/workflows/ci.yml` exists but is currently empty — the task for this repo is to configure a working GitHub Actions pipeline with build, test, and (optionally) EC2 deployment steps. The README documents the AWS deployment approach (EC2 + GitHub Actions with `AWS_ACCESS_ID`, `AWS_ACCESS_KEY`, `EC2_INSTANCE` secrets).

## Notes

- TypeScript `tsconfig.json` in both `backend/` and `frontend/` targets ES5 with strict mode.
- Prisma schema lives in `backend/prisma/schema.prisma`; run `prisma generate` after any schema change.
- Backend CORS is restricted to `localhost:3000`.
- `ManifestoBuenasPracticas.md` and `ModeloDatos.md` at the repo root document DDD conventions and the data model — read before adding new domain entities.
- `backend/src/prompts/CreateNewRoute.md` explains the pattern for adding new API routes.
