<div align="center">

# ⚡ Freelanso

### The freelance marketplace where your work speaks first.

Showcase projects, sell services, land jobs, and settle it all in a **Skill Battle**, powered by AI search and smart talent matching.

![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-8-646CFF?logo=vite&logoColor=white)
![Tailwind](https://img.shields.io/badge/Tailwind-4-06B6D4?logo=tailwindcss&logoColor=white)
![Express](https://img.shields.io/badge/Express-5-000000?logo=express&logoColor=white)
![Firebase](https://img.shields.io/badge/Firebase-Auth%20%7C%20Firestore%20%7C%20RTDB-FFCA28?logo=firebase&logoColor=black)
![Node](https://img.shields.io/badge/Node-%E2%89%A520-339933?logo=nodedotjs&logoColor=white)
![Tests](https://img.shields.io/badge/tests-Vitest%20%2B%20Emulators-6E9F18?logo=vitest&logoColor=white)

[Features](#-features) • [Tech Stack](#-tech-stack) • [Getting Started](#-getting-started) • [API](#-api-overview) • [Deployment](#-deployment) • [Contributing](#-contributing)

</div>

---

## 📖 About

**Freelanso** is a full-stack freelance marketplace built with a **React + Vite** frontend and a modular **Express** API backed by **Firebase** (Auth, Firestore, Realtime Database) and **Cloudinary** for media.

Instead of a wall of résumés, Freelanso puts **work first**: freelancers publish portfolio projects and services, clients post jobs, and the community votes in head-to-head **Skill Battles**. An AI-assisted search and a deterministic matching engine connect the right people to the right work.

<!--
📸 Add screenshots here:
![Home](./docs/screenshots/home.png)
![Explore](./docs/screenshots/explore.png)
-->

## ✨ Features

| | Feature | What it does |
|---|---|---|
| 🎨 | **Project Showcase** | Freelancers publish portfolio projects with images/video, likes, views, and comments. |
| 🛍️ | **Services** | Gig-style service listings across 7 categories with price, delivery time, and revisions. |
| 💼 | **Job Board** | Clients post jobs; freelancers apply with a cover letter and proposed rate (one application per job). |
| 📬 | **Application Pipeline** | Status state machine: `submitted → shortlisted → rejected / hired`, with applicant withdrawal. |
| ⚔️ | **Skill Battles** | Two to six creative entries go head-to-head. One vote per user, transactional counting, auto-closing on expiry. |
| 🤖 | **AI Smart Search** | Natural-language project search ranked by **Gemini 2.5 Flash**, with automatic deterministic fallback. |
| 🎯 | **Smart Matching** | Scores freelancers against a project by skills, category, budget, rating, and recency. |
| 💬 | **Real-time Messaging** | One-to-one conversations on Firebase Realtime Database with unread tracking. |
| 🔔 | **Notifications** | In-app notification feed with live updates. |
| 📊 | **Dashboard** | Listings, status counts, engagement aggregates, application counts, and rating stats. |
| 💰 | **Wallet & Withdrawals** | Integer minor-unit balances, idempotent withdrawal requests, and an append-only ledger. |
| 🎧 | **Support Desk** | Public/authenticated support tickets with honeypot spam protection and rate limiting. |
| 🔐 | **Role-based Access** | Freelancer, client, and admin roles enforced on the server. |

### 🎯 How matching works

The matching engine is deterministic and fully configurable (`server/src/config/matching.js`):

| Signal | Weight |
|---|---|
| Skill overlap | **40%** |
| Budget fit | **20%** |
| Rating | **15%** |
| Recency of activity | **15%** |
| Category match | **10%** |

### 🤖 How AI search works

- Requires authentication, **20 requests per user per UTC day**, results cached for 30 seconds.
- Sends at most **10 deterministically selected candidates** to Gemini, never raw user data or secrets.
- Retries with exponential backoff (2 retries, 9 s cap per attempt).
- If Gemini is unconfigured, slow, or returns invalid output, the API **falls back to keyword ranking** and flags `meta.degraded`. It never fabricates results.

## 🧰 Tech Stack

**Frontend**
- React 19, React Router 7, Vite 8
- Tailwind CSS 4
- GSAP for animation, Lucide for icons
- Firebase Web SDK (Auth + Realtime Database)

**Backend**
- Node.js 20+, Express 5 (ES modules)
- Firebase Admin SDK (Auth, Firestore, Realtime Database)
- Zod validation, Helmet, CORS, compression, `express-rate-limit`
- Pino structured logging with request IDs
- Cloudinary signed uploads, Google Gemini (optional)

**Quality & DevOps**
- Vitest + Supertest, Firebase Emulator integration & security-rules tests
- OpenAPI spec with a route-parity checker
- ESLint, Prettier, Husky pre-commit hooks
- GitHub Actions CI with **Gitleaks** secret scanning and `npm audit`
- Docker + Google Cloud Run, Vercel for the frontend

## 🗂️ Project Structure

```
Freelanso/
├── src/                      # React frontend
│   ├── components/           # Navbar, Sidebar, ProjectCard, BattleCard, HireModal, ...
│   ├── pages/                # Home, Explore, BrowseJobs, Messages, Profile, Dashboard, ...
│   └── lib/                  # api client, auth, firebase, cloudinary, hooks
├── server/                   # Express API
│   ├── src/
│   │   ├── config/           # env, cors, firebase, logger, matching weights
│   │   ├── middleware/       # auth, requireRole, validate, rateLimit, errorHandler, ...
│   │   ├── modules/          # feature modules (routes → controller → service → repository)
│   │   │   ├── applications/ battles/    conversations/ dashboard/
│   │   │   ├── jobs/         matches/    notifications/ projects/
│   │   │   ├── search/       services/   support/       uploads/
│   │   │   └── users/        withdrawals/ health/ me/ meta/
│   │   └── scripts/          # seed, migrate, set-role, load test, openapi check
│   ├── tests/                # unit, integration & emulator tests
│   ├── deploy/               # Cloud Run service definition
│   ├── openapi.yaml          # API specification
│   ├── firestore.rules       # Firestore security rules
│   └── database.rules.json   # Realtime Database rules
├── firestore.rules
├── storage.rules
└── vercel.json
```

## 🚀 Getting Started

### Prerequisites

- **Node.js 20+** and npm
- **Java 21+** (for Firebase emulators)
- A **Firebase** project (Auth, Firestore, Realtime Database enabled)
- *(Optional)* a **Cloudinary** account and a **Gemini API key**

### 1. Clone & install

```bash
git clone https://github.com/<your-username>/Freelanso.git
cd Freelanso

# Frontend
npm ci

# Backend
cd server
npm ci
cd ..
```

### 2. Configure environment

**Frontend**: copy `.env.example` to `.env`:

```env
VITE_API_URL=http://localhost:5000/api
```

> Also add your Firebase Web SDK config in `src/lib/firebase.js` (or via `VITE_FIREBASE_*` variables, depending on your setup).

**Backend**: copy `server/.env.example` to `server/.env`:

```env
NODE_ENV=development
PORT=5000
CORS_ALLOWED_ORIGINS=http://localhost:5173

# Firebase
FIREBASE_PROJECT_ID=your-firebase-project-id
FIREBASE_DATABASE_URL=
GOOGLE_APPLICATION_CREDENTIALS=      # path to a service account JSON OUTSIDE the repo

# Cloudinary (optional, configure all together)
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=
CLOUDINARY_PROFILE_UPLOAD_PRESET=
CLOUDINARY_PROJECT_UPLOAD_PRESET=
CLOUDINARY_BATTLE_UPLOAD_PRESET=

# AI search (optional, set all three or leave blank for keyword-only)
AI_PROVIDER=gemini
AI_MODEL=gemini-2.5-flash
AI_API_KEY=

# Hardening
LOG_LEVEL=info
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX=120
TRUST_PROXY=false
REQUEST_TIMEOUT_MS=30000

# Wallet (amounts in minor units / paise)
WITHDRAWAL_MIN_MINOR=10000
WITHDRAWAL_MAX_MINOR=1000000
WALLET_CURRENCY=INR
```

> ⚠️ **Never commit** `.env` files or service-account JSON. Use Application Default Credentials or keep the key outside the repository.

### 3. Run locally (with emulators)

Open three terminals:

```bash
# Terminal 1: Firebase emulators
cd server
npx firebase emulators:start --project demo-freelanso-test --only auth,firestore,database

# Terminal 2: API
cd server
npm run dev

# Terminal 3: Frontend
npm run dev
```

The app is now at **http://localhost:5173** and the API at **http://localhost:5000**.

### 4. Seed demo data *(development / emulator only)*

```bash
cd server
npm run seed:services     # demo services
npm run seed:jobs         # demo jobs + client profiles
npm run set-role          # assign a role (freelancer / client / admin) to a user
```

## 🧪 Testing & Quality

```bash
# Frontend
npm run lint
npm run build

# Backend (run from /server)
npm run lint               # ESLint
npm test                   # unit tests
npm run test:emu           # unit + integration tests against Firebase emulators
npm run test:rules         # Firestore & RTDB security-rules tests
npm run test:indexes       # verify every query has a Firestore index
npm run test:coverage      # coverage report
npm run openapi:check      # OpenAPI ↔ Express route parity
npm run load:test:emu      # load test with autocannon
```

CI (`.github/workflows/ci.yml`) runs secret scanning, builds, lint, OpenAPI checks, dependency audits, and the full emulator test suite on every PR and push to `main`.

## 🔌 API Overview

Base URL: `http://localhost:5000/api`. Full spec in [`server/openapi.yaml`](./server/openapi.yaml).

| Area | Endpoints |
|---|---|
| **Health** | `GET /healthz` · `GET /readyz` |
| **Auth & Profile** | `GET /me` · `GET/PUT /users/:uid` · `POST /users/onboarding` |
| **Projects** | `GET/POST /projects` · `GET/PUT/DELETE /projects/:id` · `POST /projects/:id/like` · `GET/POST /projects/:id/comments` |
| **Services** | `GET/POST /services` · `GET/PUT/DELETE /services/:id` · `GET /meta/categories` |
| **Jobs** | `GET/POST /jobs` · `GET/PUT/DELETE /jobs/:id` · `POST /jobs/:id/applications` |
| **Applications** | `GET /applications/me` · `PATCH /applications/:id/status` |
| **Battles** | `GET/POST /battles` · `GET /battles/:id` · `POST /battles/:id/vote` |
| **Search & Match** | `POST /search/smart` · `POST /matches/score-all` |
| **Messaging** | `POST /conversations` |
| **Wallet** | `GET /wallet` · `POST /withdrawals` · `GET /withdrawals/me` |
| **Support** | `POST /support/tickets` · `GET /support/tickets/me` |
| **Uploads** | `POST /uploads/sign` |
| **Dashboard** | `GET /dashboard` |

**Highlights**
- Cursor-based pagination everywhere, with five sort modes (`newest`, `oldest`, `price_asc`, `price_desc`, `popular`).
- Consistent response envelopes and validated inputs (Zod).
- Transactions for likes, votes, applications, and wallet operations.
- `Idempotency-Key` support on withdrawals and support tickets.

## 🔒 Security

- Firebase ID token verification (revocation checked) on protected routes
- Role-based guards (`freelancer`, `client`, `admin`)
- Helmet, strict CORS allow-list, HTTPS enforcement in production
- Per-IP/per-user rate limiting and request timeouts
- Signed-only Cloudinary uploads with server-enforced presets and per-user folders
- Firestore, Realtime Database, and Storage security rules, covered by emulator tests
- Honeypot + rate limiting on public forms
- Gitleaks secret scanning in CI

## ☁️ Deployment

**Frontend → Vercel**
- `vercel.json` is included (static build from `dist/`, SPA rewrites).
- Set `VITE_API_URL` to your deployed API's HTTPS `/api` URL.

**Backend → Google Cloud Run**
- Build with the provided `server/Dockerfile` (Node 20 Alpine, non-root user, port `8080`).
- Service definition in `server/deploy/cloudrun.service.yaml`.
- Set `CORS_ALLOWED_ORIGINS` to your exact frontend origin(s). No wildcards.
- Deploy Firestore indexes and rules with the Firebase CLI:
  ```bash
  cd server
  firebase deploy --only firestore:rules,firestore:indexes,database
  ```

See [`server/RUNBOOK.md`](./server/RUNBOOK.md) for operations and [`server/ARCHITECTURE.md`](./server/ARCHITECTURE.md) for the architecture and security review.

## 📚 More Documentation

- [Server setup & endpoint docs](./server/README.md)
- [Architecture & security review](./server/ARCHITECTURE.md)
- [Operations runbook](./server/RUNBOOK.md)
- [OpenAPI specification](./server/openapi.yaml)
- [Skill Battles guide](./SKILL_BATTLES_GUIDE.md)
- [Feature → API mapping](./FEATURE_TO_API_MAPPING.md)

## 🗺️ Roadmap

- [ ] Stripe Connect payouts (withdrawals are currently approved manually; no live payout is executed)
- [ ] Earnings credit flow and deposit ledger
- [ ] Reviews & ratings after hire
- [ ] Email notifications
- [ ] Additional AI providers behind the existing provider interface

## 🤝 Contributing

Contributions are welcome!

1. Fork the repo and create a branch: `git checkout -b feat/amazing-feature`
2. Make your changes and add tests
3. Run `npm run lint` and `cd server && npm run test:emu`
4. Commit (Husky runs pre-commit checks) and open a Pull Request

## 📄 License

Add your license here (e.g. MIT) and include a `LICENSE` file in the repository root.

---

<div align="center">

**Built with ⚡ by the Freelanso team**

If you like this project, consider giving it a ⭐

</div>
