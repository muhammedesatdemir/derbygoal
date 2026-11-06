# DerbyGoal Architecture & Engineering Documentation

[![Website](https://img.shields.io/badge/Website-derbygoal.com-10b981?style=for-the-badge&logo=google-chrome&logoColor=white)](https://derbygoal.com/)
[![Google Play](https://img.shields.io/badge/Google_Play-DerbyGoal-34d399?style=for-the-badge&logo=google-play&logoColor=white)](https://play.google.com/store/apps/details?id=com.derbygoal.app)
[![Node Runtime](https://img.shields.io/badge/Node.js-%3E%3D20-339933?style=for-the-badge&logo=node.js&logoColor=white)](https://nodejs.org/)
[![Turborepo](https://img.shields.io/badge/Monorepo-Turborepo-EF4444?style=for-the-badge&logo=turborepo&logoColor=white)](https://turbo.build/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.5-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)

---

## 1. Executive Summary & Architectural Vision

**DerbyGoal** is a production-grade, multi-platform football trivia and strategy card gaming ecosystem. Designed as a unified TypeScript monorepo, it powers concurrent, real-time multiplayer, hotseat, and offline algorithmic bot sessions across Web (Next.js 14 App Router) and Mobile (React Native / Expo 51+).

The core architectural requirement is **Zero-Unnecessary-Server-Cost & High Fault Tolerance**:
1. **Client-Deterministic Simulation:** Core game rules, turns, bot personalities, dice physics, and match state transitions execute via seeded PRNG engines on client runtimes, guaranteeing 100% logic and visual parity while imposing zero compute load on serverless tiers.
2. **Strict Fault Isolation & Fallback Chains:** In-flight network partitions, missing remote telemetry, or payment gateway latencies gracefully degrade to offline state machines, deterministic mock sets, and memory-safe queues.
3. **Optimistic & Idempotent Transactions:** Player move validations and economy actions leverage transactional state mutations, preventing race conditions, replay attacks, and duplicate deductions.

---

## 2. High-Level System Architecture

The following diagram details the clean architecture layers, data boundaries, and third-party orchestration:

```mermaid
graph TD
    subgraph ClientTiers ["Client Applications (Presentation Layer)"]
        WebClient["Web Application (Next.js 14 / Tailwind / Framer Motion)"]
        MobileClient["Mobile Application (React Native / Expo / Reanimated)"]
    end

    subgraph CoreDomain ["Domain & Engine Layer (Platform-Agnostic)"]
        GameEngine["@futbol-kart/game-engine (Seeded PRNG & Deterministic State Machine)"]
        QuestionTemplates["@futbol-kart/question-templates (Metric Math & Localization Synthesizer)"]
        SharedTypes["@futbol-kart/shared-types (Contracts, State Schemas, Tile Models)"]
    end

    subgraph ServiceLayer ["API & Edge Gateway (Next.js App Router)"]
        AuthService["Auth & Session Validation (Better Auth / Secure Tokens)"]
        MatchmakingHub["Matchmaking & Lobby Orchestration"]
        MoveDispatcher["Idempotent Move Resolution Gateway"]
        WebhooksHandler["Asynchronous Webhook Receivers (Stripe / RevenueCat)"]
    end

    subgraph InfrastructureLayer ["Data & Real-Time Infrastructure Layer"]
        NeonDB[("Neon Serverless PostgreSQL (Drizzle ORM / HTTP Driver)")]
        AblyRealtime["Ably Real-time Message Broker (Token-Gated Channels)"]
        StorageEngine["Edge Asset Storage & Dynamic CDN"]
    end

    WebClient --> SharedTypes
    WebClient --> GameEngine
    MobileClient --> SharedTypes
    MobileClient --> GameEngine

    WebClient -- "HTTP / REST APIs" --> ServiceLayer
    MobileClient -- "HTTP / REST APIs" --> ServiceLayer

    WebClient -. "WebSocket Delta Sync" .-> AblyRealtime
    MobileClient -. "WebSocket Delta Sync" .-> AblyRealtime

    ServiceLayer --> NeonDB
    ServiceLayer --> AblyRealtime
    GameEngine --> QuestionTemplates
```

---

## 3. Monorepo Organization & Module Boundaries

The project adheres to strict Clean Architecture principles, enforcing unidirectional dependencies between presentation apps and shared domain packages.

```
futbol-kart/
├── apps/
│   ├── mobile/                 # Cross-platform Mobile App (Expo SDK 51, React Native)
│   │   ├── src/
│   │   │   ├── components/     # High-performance native widgets & pitch components
│   │   │   ├── screens/        # Mode orchestration & game boards
│   │   │   ├── lib/            # Native services (IAP, Haptics, Audio Engine, State)
│   │   │   └── i18n/           # 17-language multi-locale dictionaries
│   │   └── package.json
│   └── web/                    # Universal Web Application (Next.js 14 App Router)
│       ├── src/
│       │   ├── app/            # Edge API endpoints, layouts, route handlers
│       │   ├── components/     # Accessible UI, CSS Grid / Canvas pitches
│       │   └── lib/            # Server-side DB proxies, Ably tokens, SFX
│       └── package.json
├── packages/
│   ├── game-engine/            # Platform-agnostic state machine, PRNG, bots
│   ├── question-templates/     # Metric question builders & multi-lingual syntax
│   ├── shared-types/           # Pure TypeScript models, DTOs, and event signatures
│   └── db/                     # Drizzle ORM schema, migrations, connection proxy
├── turbo.json                  # Turborepo task pipeline orchestration
└── package.json                # Workspace configuration (pnpm v10+)
```

---

## 4. Key Architectural Patterns & Resilience

### 4.1. Deterministic Pseudo-Random Number Generation (PRNG)
To eliminate synchronous server round-trips for multiplayer dice rolls, board moves, bot decisions, and tile rewards, the game utilizes linear congruential / splitmix-derived seed generators (`createPRNG`):
* Sessions are initialized with a synchronized seed hash (e.g., `seed = hash(matchId, roundNumber)`).
* Bot algorithms predict moves, select cards, and compute question answers with simulated "human thought delays" without calling any remote AI endpoints.
* Ensures 100% deterministic replayability and client-to-client consistency.

### 4.2. Database Connection Virtualization & Lazy Proxies
The database access layer (`@futbol-kart/db`) connects to Neon Serverless PostgreSQL via HTTP pipelines rather than long-lived TCP connection pools:
* **Cold-Start Elimination:** No idle pool overhead in serverless edge functions.
* **Lazy Proxy Initialization:** Prevents deployment and build-step crashes when `DATABASE_URL` is withheld during static pre-rendering passes. Modules only establish network handles upon actual SQL dispatch.

### 4.3. Dual Payment Gateway Isolation with Idempotency
* **Mobile Tiers:** RevenueCat SDK handles Google Play Store cryptographic receipt validation, using localized currency caches as offline fallbacks (`api.revenuecat.com` -> disk cache fallback).
* **Web Tiers:** Stripe Checkout with server-side signature verification.
* **Webhook Idempotency:** Webhook handlers process incoming event UUIDs inside transactional locks to guarantee that double-deliveries never result in duplicate currency or premium status assignments.

---

## 5. API Endpoint Specifications

The following table details the primary HTTP endpoints provided by the edge service gateway:

| Method | Endpoint Path | Authentication | Description | Fault Tolerance / Fallback |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/api/matchmaking` | Session Bearer | Queues player for rating-matched duel | Falls back to offline bot matchmaker if no human matches within queue timeout |
| `GET` | `/api/match/[matchId]` | Session Bearer | Fetches authoritative match state snapshot | Memory-cached state returned if DB read experiences transit latency |
| `POST` | `/api/match/[matchId]/move` | Session Bearer | Idempotent submission of player board action | Transaction rejected on timestamp collision; client re-syncs state |
| `POST` | `/api/match/[matchId]/ably-token` | Session Bearer | Issues capability-scoped transient real-time token | Degrades to HTTP long-polling if WebSocket upgrade handshake fails |
| `GET` | `/api/leaderboard` | Public / Session | Retrieves global and seasonal Elo leaderboards | Stale-while-revalidate cached; returns in-memory snapshot on DB partition |
| `GET` | `/api/friends` | Session Bearer | Retrieves friend list, presence, and challenges | Degrades to cached friend store |
| `POST` | `/api/webhooks/revenuecat` | Webhook Signature | Ingestion of Google Play subscription changes | Idempotent event deduplication via signature & event ID checks |
| `POST` | `/api/webhooks/stripe` | Stripe Signature | Web checkout completion and billing sync | Cryptographically validated; duplicate transactions safely ignored |

---

## 6. Developer & Contributor Guide

### 6.1. Prerequisites
- **Node.js:** `>= 20.0.0`
- **Package Manager:** `pnpm >= 10.0.0` (Corepack recommended: `corepack enable`)
- **Native Tools (Optional for Mobile):** Android Studio / Xcode, Expo CLI

### 6.2. Environment Configuration
Create `.env.local` inside `apps/web/` and `.env` inside `apps/mobile/`:

```env
# Database & Persistence (Neon Serverless PostgreSQL)
DATABASE_URL="postgresql://user:password@ep-sample-pool.eu-central-1.aws.neon.tech/neondb?sslmode=require"

# Real-time WebSocket Gateway (Ably)
ABLY_API_KEY="your-ably-api-key"

# Client Application URLs
NEXT_PUBLIC_APP_URL="https://derbygoal.com"
```

### 6.3. Development Workflow

1. **Install Dependencies Across Monorepo:**
   ```bash
   pnpm install
   ```

2. **Run Full-Stack Type Verification:**
   ```bash
   pnpm typecheck
   ```

3. **Launch Web Environment:**
   ```bash
   pnpm dev
   ```

4. **Launch Mobile Environment (Expo Metro Bundler):**
   ```bash
   pnpm mobile:start
   ```

5. **Lint and Code Quality Validation:**
   ```bash
   pnpm lint
   ```

---

## 7. Official Distribution & Play Venues

* **Web Application:** [https://derbygoal.com/](https://derbygoal.com/)
* **Android Google Play Store:** [DerbyGoal on Google Play](https://play.google.com/store/apps/details?id=com.derbygoal.app)

