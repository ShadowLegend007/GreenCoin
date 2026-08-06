# GreenCoin Backend

The backend API server for the GreenCoin e-waste recycling platform. Built with **Express 5**, **TypeScript**, **MongoDB (Mongoose)**, and **Zod** for request validation.

---

## GreenCoin Backend Architecture

**Goal**
Build a secure, robust, and scalable RESTful API server for the GreenCoin e-waste recycling platform. The backend handles user authentication, manages the strict lifecycle of e-waste pickups, and powers a decoupled gamification engine to drive user engagement.

The architecture strictly enforces separation of concerns by isolating business logic (Services) from HTTP transport (Controllers) and validation (Zod Middlewares).

### High-Level Architecture
```text
                     CLIENT REQUESTS
                           │
                           ▼
                  Express Router (Express 5)
                           │
        ┌──────────────────┼────────────────────┐
        │                  │                    │
   Zod Validation    JWT Auth Guard       RBAC Middleware
   (Req.body)       (Extracts user)      (Checks Roles)
        │                  │                    │
        └──────────────────┼────────────────────┘
                           │
                     Controllers
               (Extracts params/body)
                           │
                           ▼
                       Services
               (Core Business Logic)
                           │
        ┌──────────────────┼────────────────────┐
        │                  │                    │
  State Machine       Rewards Client       Mongoose Models
(Enforces Rules)    (External APIs)       (MongoDB Access)
        │                  │                    │
        └──────────────────┼────────────────────┘
                           │
                           ▼
                   Event Dispatcher ───► Gamification Event Bus
```

### Scalability & Design Principles
- **Strict Data Validation:** Zod intercepts bad payloads at the routing layer, guaranteeing the controller and service only process safe, correctly-typed data.
- **Centralized Error Handling:** All errors bubble up to a single Express middleware that formats them consistently `{ success: false, error: "ERROR_CODE", message: "..." }`, hiding stack traces from clients.
- **State Machine Integrity:** Hardcoding state transitions in an isolated map guarantees that no rogue API call can mutate a pickup into an illegal state.
- **Event-Driven Decoupling:** Core operations (like picking up a laptop) have zero dependencies on engagement operations (like awarding XP). If the gamification engine crashes, core business workflows remain unaffected.
- **Structured Logging:** A centralized logger ensures production logs are parsable (JSON/ISO timestamps) and easily suppressed during test runs to reduce noise.

---

## Detailed Implementation Breakdown

This backend implements the **Pickup Module**, **Authentication & Users Module**, and the **Gamification Engine** — forming the core workflows of the GreenCoin ecosystem. Below is a breakdown of everything that was done.

### 1. Pickup Module Architecture

**Goal**
Manage the core e-waste recycling lifecycle—from request to verification—while enforcing strict role-based access controls and a forward-only state machine to prevent inconsistent states.

#### High-Level Architecture
```text
                     USER ACTIONS (App/Web)
                           │
        ┌──────────────────┼────────────────────┐
        │                  │                    │
  Request Pickup      Accept Pickup       Verify & Complete
  (Role: User)     (Role: Collector)      (Role: Admin)
        │                  │                    │
        └──────────────────┼────────────────────┘
                           │
                   Pickup Controller
               (Zod Validation + Auth)
                           │
                           ▼
                     Pickup Service
              (Enforces State Machine)
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
 Database (Mongoose) Rewards Client  Event Dispatcher
   (Save State)    (Trigger Rewards)  (Gamification)
```

#### Folder Structure
```text
pickup/
│
├── pickup.routes.ts              # Route definitions & middleware chain
├── pickup.controller.ts          # Request/Response orchestration
├── pickup.service.ts             # Core CRUD and status logic
├── pickup.model.ts               # Mongoose schemas for Pickups & Devices
├── pickup.validation.ts          # Zod schemas for input validation
├── pickup-state-machine.ts       # Forward-only transition rules
├── rewards-client.ts             # External HTTP client for Rewards service
│
├── collection-center.routes.ts   # Collection center routes
├── collection-center.controller.ts # Collection center orchestration
├── collection-center.service.ts  # Collection center logic
└── collection-center.validation.ts # Zod schemas for collection centers
```

#### State Machine Pipeline
A pickup request moves through a strict, forward-only lifecycle mapped in `pickup-state-machine.ts`:
```text
Requested → Accepted → Picked → Delivered → Verified ─┬─→ Reward Generated
                                                      └─→ Verification Failed
```
- **Only valid forward transitions are allowed.** Bypassing a step (e.g. `Requested → Delivered`) throws a `400 INVALID_TRANSITION` error.
- **`Verification Failed`** is reached only when the Rewards service handoff fails, preventing pickups from getting stuck in limbo due to network errors.

#### Business Rules Enforced
- **Collector Assignment**: A collector must be *assigned* to a pickup before updating its status.
- **Strict Role Isolation**: A different collector cannot update a pickup they are not assigned to (`403 FORBIDDEN_NOT_ASSIGNED_COLLECTOR`).
- **No Double Acceptance**: A pickup that already has a collector cannot be accepted again (`403 FORBIDDEN_ALREADY_ASSIGNED`).
- **Data Isolation**: Users can only view their own pickups. Collectors can only view pickups that are `Requested` or explicitly assigned to them.
- **Param Validation**: Prevents 500 crashes (`CastError`) by enforcing valid MongoDB ObjectIds format on all route parameters.

#### Database Collections
- **Pickups (`pickups`)**: Tracks state, `pickupTime`, `userId`, `collectorId`, and embedded `deviceId`.
- **Devices (`devices`)**: Stores the physical item's `category` and `weight`.
- **Collection Centers (`collection_centers`)**: Admin-managed physical drop-off locations (`name`, `location`).

#### API Layer
```text
# Pickups
POST   /api/v1/pickups                 (User)
GET    /api/v1/pickups                 (User/Collector/Admin)
GET    /api/v1/pickups/:id             (User/Collector/Admin)
PATCH  /api/v1/pickups/:id/accept      (Collector)
PATCH  /api/v1/pickups/:id/status      (Collector)
PATCH  /api/v1/pickups/:id/verify      (Admin)

# Collection Centers
GET    /api/v1/collection-centers      (Authenticated)
POST   /api/v1/collection-centers      (Admin)
```

#### Scalability Principles
- **State Machine Integrity**: Decoupling the transition logic into a pure static class makes testing the boundaries straightforward and guarantees safety.
- **Delegated Triggers**: The module uses `rewards-client.ts` to trigger external reward systems and `dispatchEvent` for the gamification engine, offloading heavy processing.
- **Zod First**: Validating schemas directly inside the route definitions guarantees controllers never handle malformed bodies or query parameters.

### 5. Authentication & Users

Full JWT-based Authentication and User Management have been implemented:

- **Auth Routes** → [`auth.routes.ts`](src/auth/auth.routes.ts): Provides `/register`, `/login`, and `/logout` endpoints. Uses `bcrypt` for secure password hashing and `jsonwebtoken` for issuing JWTs.
- **User Routes** → [`user.routes.ts`](src/users/user.routes.ts): Provides profile management with endpoints for `GET /me`, `PATCH /me`, and admin-only routes like `GET /:id` and `GET /`.
- **Middleware** → [`auth.middleware.ts`](src/middlewares/auth.middleware.ts) validates the JWT and injects `req.user`. Additionally, `rbac.middleware.ts` provides role-based access control.

### 5.5. Gamification Engine Architecture

**Goal**
Build a fully modular, scalable, event-driven gamification engine that remains independent from the core application.
The engine should never directly perform business operations (pickup, authentication, scanning, etc.).
Instead, it listens to events generated by other modules and computes:
- GreenCoin rewards
- Badges
- Levels
- XP
- Leaderboards
- Challenges
- Wallet transactions
- Reward redemption eligibility
- User statistics

This allows the gamification module to be plugged into any backend with minimal coupling.

#### High-Level Architecture
```text
                     USER ACTIONS
                           │
        ┌──────────────────┼────────────────────┐
        │                  │                    │
 Device Scan         Pickup Completed      Referral Joined
        │                  │                    │
        └──────────────────┼────────────────────┘
                           │
                  Business Modules
                           │
                  Emit Domain Events
                           │
                           ▼
               Gamification Event Bus
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
 Reward Engine      Badge Engine       XP Engine
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                     Level Engine
                           │
                           ▼
                    Wallet Engine
                           │
        ┌──────────────────┼────────────────────┐
        │                  │                    │
 Leaderboard        Challenge Engine      Notification
        │                  │                    │
        └──────────────────┼────────────────────┘
                           ▼
                    MongoDB Database
```

#### Folder Structure
```text
gamification/
│
├── engine/
│   ├── reward_engine.js
│   ├── xp_engine.js
│   ├── badge_engine.js
│   ├── level_engine.js
│   ├── leaderboard_engine.js
│   ├── wallet_engine.js
│   ├── streak_engine.js
│   ├── challenge_engine.js
│   ├── notification_engine.js
│   └── gamification_service.js
│
├── events/
│   ├── event_bus.js
│   ├── event_dispatcher.js
│   └── event_types.js
│
├── rules/
│   ├── reward_rules.js
│   ├── badge_rules.js
│   ├── level_rules.js
│   ├── streak_rules.js
│   └── challenge_rules.js
│
├── wallet/
│   ├── wallet_model.js
│   ├── transaction_model.js
│   └── wallet_service.js
│
├── leaderboard/
│   ├── leaderboard_model.js
│   └── leaderboard_service.js
│
├── badges/
│   ├── badge_model.js
│   └── user_badge_model.js
│
├── rewards/
│   ├── reward_catalog.js
│   ├── redemption_service.js
│   └── coupon_service.js
│
├── models/
│   ├── user_stats.js
│   ├── activity_log.js
│   └── gamification_profile.js
│
└── api/
    ├── wallet_routes.js
    ├── reward_routes.js
    ├── leaderboard_routes.js
    ├── badge_routes.js
    └── profile_routes.js
```

#### Event-Driven Pipeline
Every business module emits an event.
Example:
```text
Pickup Completed
        │
        ▼
Emit Event
{
    type: "PICKUP_COMPLETED",
    userId,
    collectorId,
    weight,
    category,
    timestamp
}
↓
Gamification Service receives event
↓
Reward Engine calculates coins
↓
XP Engine calculates XP
↓
Badge Engine checks achievements
↓
Level Engine updates level
↓
Wallet Engine credits wallet
↓
Leaderboard recalculates score
↓
Notification Engine notifies user
↓
Save everything to database
```

#### Supported Events
- `USER_REGISTERED`
- `DEVICE_SCANNED`
- `EWASTE_SUBMITTED`
- `PICKUP_COMPLETED`
- `PICKUP_VERIFIED`
- `REFERRAL_SUCCESS`
- `REWARD_REDEEMED`
- `CHALLENGE_COMPLETED`
- `STREAK_UPDATED`
- `DAILY_LOGIN`
- `PROFILE_COMPLETED`
- `COLLECTOR_REVIEWED`
- `CAMPAIGN_COMPLETED`
- `SPECIAL_EVENT`

Every future feature simply emits one of these events.

#### Reward Engine
Responsible for:
`Receive event` ↓ `Find reward rule` ↓ `Calculate base reward` ↓ `Apply multipliers` ↓ `Return final coins`

**Reward Formula**
`Reward = Base Coins × Weight Multiplier × Category Multiplier × Campaign Multiplier × Streak Multiplier × Bonus Multiplier`

**Example**
- Laptop Base = 150
- Weight Bonus = 1.3
- Campaign = 2x
- Weekend = 1.2
- **Total**: `150 × 1.3 × 2 × 1.2 = 468 Coins`

#### XP Engine
Coins ≠ XP. XP measures engagement.
- **Daily Login**: +10 XP
- **Pickup**: +100 XP
- **Referral**: +80 XP
- **Review**: +20 XP
- **Campaign**: +150 XP

XP drives Levels.

#### Level Engine
- **Level 1**: 0 XP
- **Level 2**: 200 XP
- **Level 3**: 500 XP
- **Level 4**: 900 XP
- **Level 5**: 1500 XP

Benefits: Higher level ↓ Higher badge rarity ↓ Special campaigns ↓ Exclusive rewards ↓ Priority rankings.

#### Wallet Engine
Wallet stores: Current Balance, Lifetime Coins, Coins Earned, Coins Redeemed, Pending Coins.
Every transaction: Credit, Debit, Expiry, Bonus, Campaign Reward, Redemption.

**Transaction schema:** `transaction { id, userId, type, coins, reason, referenceId, timestamp }`

Nothing updates balance directly. Everything creates transactions.
`Wallet balance = Sum(all transactions)`

#### Badge Engine
Checks rules after every event.
Example: `First Device` ↓ `Recycle 1 item` ↓ `Badge Unlocked`

Example badges: First Step, Eco Beginner, Recycler, Eco Warrior, Green Hero, Collector Friend, Referral Master, Carbon Saver, Earth Protector, Tech Recycler, Champion, Legend.
Badges have tiers: Bronze, Silver, Gold, Platinum, Diamond.

#### Streak Engine
Tracks continuous actions: Daily Login, Pickup, Weekly Recycling, Monthly Recycling.
Example:
- 3 day streak ↓ +20 Coins
- 7 day streak ↓ +100 Coins
- 30 day streak ↓ Special Badge

#### Leaderboard Engine
Leaderboards: Global, City, College, Company, Campaign, Friends.
Ranking Score Example: `Score = XP + Coins × 0.1 + Badges × 50 + Challenges × 100`
Leaderboard updates: Realtime, Every 5 minutes, or Nightly batch.

#### Challenge Engine
Creates missions.
Example: `Recycle 5 Devices` ↓ `Reward: 300 Coins`
Campaign example: `Earth Day: Recycle 3 kg` ↓ `Reward: Badge + 500 Coins`
Challenge lifecycle: `Created` ↓ `Assigned` ↓ `Started` ↓ `Completed` ↓ `Reward Issued`

#### Reward Redemption Pipeline
`User` ↓ `Browse Rewards` ↓ `Check Wallet Balance` ↓ `Redeem` ↓ `Wallet Debit` ↓ `Coupon Generated` ↓ `Notify User` ↓ `Transaction Stored`

Reward Types: Amazon Voucher, Flipkart Coupon, Boat Coupon, Croma Coupon, Donation, Plant Tree, CSR Rewards, Event Tickets, Premium Badges, Campus Merchandise.

#### Notification Engine
Triggered after every achievement.
Example: `Congratulations! You earned 250 GreenCoins + Eco Warrior Badge + Reached Level 5`
Supports: Push Notification, Email, In-App, SMS.

#### Database Collections
`wallets`, `wallet_transactions`, `badges`, `user_badges`, `levels`, `leaderboards`, `user_statistics`, `reward_catalog`, `reward_redemptions`, `challenges`, `user_challenges`, `activity_logs`, `campaigns`, `gamification_profiles`

#### Complete Event Flow
`User Recycles Laptop` ↓ `Pickup Verified` ↓ `Emit Event` ↓ `Reward Engine` ↓ `XP Engine` ↓ `Badge Engine` ↓ `Level Engine` ↓ `Wallet Credit` ↓ `Leaderboard Update` ↓ `Challenge Check` ↓ `Notification` ↓ `Save Transactions` ↓ `Frontend Receives Updated Wallet` ↓ `User Sees: +450 Coins, New Badge, Level Up, Leaderboard Rank`

#### API Layer
```
GET    /wallet
GET    /wallet/history
GET    /leaderboard
GET    /badges
GET    /profile/gamification
GET    /challenges
POST   /redeem
GET    /rewards
GET    /levels
GET    /statistics
```

#### Scalability Principles
- **Event-driven**: Business modules emit events; gamification reacts asynchronously.
- **Stateless engines**: Reward, XP, badge, level, and streak engines are pure calculation services, making them easy to test and scale horizontally.
- **Configuration-driven rules**: Store reward formulas, badge criteria, level thresholds, and challenge definitions in configuration/database rather than hardcoding logic.
- **Append-only wallet ledger**: Never mutate balances directly; derive wallet balance from immutable transactions for auditability.
- **Independent APIs**: Frontend consumes gamification endpoints without coupling to business services.
- **Pluggable integrations**: New events, campaigns, rewards, or notification channels can be added without modifying the core engine.

This architecture cleanly separates business logic (pickup, verification, scanning) from engagement logic (coins, XP, badges, leaderboards), making the gamification engine reusable, maintainable, and ready for future growth.

### 6. Error Handling

A centralized [`error.middleware.ts`](src/middlewares/error.middleware.ts) catches all errors thrown by controllers/services and returns a consistent JSON response:

```json
{
  "success": false,
  "error": "ERROR_CODE",
  "message": "Human-readable description"
}
```

Error codes used throughout the API:

- `UNAUTHORIZED` — Missing or invalid auth token
- `FORBIDDEN` — Role-based access denied
- `FORBIDDEN_ALREADY_ASSIGNED` — Pickup already has a collector
- `FORBIDDEN_NOT_ASSIGNED_COLLECTOR` — Collector not assigned to this pickup
- `NOT_FOUND` — Resource not found
- `INVALID_TRANSITION` — Invalid state machine transition
- `VALIDATION_ERROR` — Zod validation failure
- `REWARDS_HANDOFF_FAILED` — Triggering rewards via external service failed
- `INTERNAL_SERVER_ERROR` — Unhandled errors

### 7. Structured Logging

A lightweight [`logger.ts`](src/utils/logger.ts) utility provides `debug`, `info`, `warn`, and `error` log levels with ISO timestamps. Logs are automatically suppressed during test execution (`NODE_ENV=test`) to keep output clean.

### 8. Database Models & Indexes

Three Mongoose models defined in [`pickup.model.ts`](src/pickup/pickup.model.ts):

| Model | Fields |
|-------|--------|
| **Pickup** | `status`, `pickupTime`, `userId`, `collectorId`, `deviceId`, `createdAt`, `updatedAt` |
| **Device** | `category`, `weight` |
| **CollectionCenter** | `name`, `location` |

Database indexes are set on `Pickup` for fast querying:

- `userId` — for user-specific pickup listing
- `collectorId` — for collector-specific filtering
- `status` — for status-based filtering

### 9. Test Suite

- **Test Suite** → [`pickup.test.ts`](tests/pickup.test.ts): Contains 14 Jest/Supertest tests spanning the complete lifecycle of Pickups. Validates restrictions on User vs Collector vs Admin endpoints, validates forward-only status updates, and comprehensively tests `verifyPickup` with Rewards successful/failure cases using mocked services.
- Ensures logic is heavily protected against regression. 

---

## Project Structure

```
backend/
├── src/
│   ├── index.ts                          # Express app setup, routing & execution
│   ├── config/
│   │   └── db.ts                         # MongoDB connection setup
│   ├── middlewares/
│   │   ├── auth.middleware.ts            # JWT authentication middleware
│   │   ├── error.middleware.ts           # Centralized error handler
│   │   └── rbac.middleware.ts            # Role-based access control
│   ├── auth/
│   │   ├── auth.controller.ts            # Register & login request handlers
│   │   ├── auth.routes.ts                # Auth route definitions
│   │   ├── auth.service.ts               # Password hashing & auth logic
│   │   ├── auth.validation.ts            # Zod schemas for auth inputs
│   │   └── jwt.util.ts                   # Token generation and verification
│   ├── users/
│   │   ├── user.controller.ts            # User profile request handlers
│   │   ├── user.model.ts                 # User schema
│   │   ├── user.routes.ts                # User routes
│   │   └── user.service.ts               # User business logic
│   ├── gamification/                     # Gamification engine (pub/sub)
│   │   ├── api/                          # Gamification endpoints
│   │   ├── badges/                       # Badge logic
│   │   ├── engine/                       # Core event processing
│   │   ├── events/                       # Dispatcher and definitions
│   │   ├── leaderboard/                  # Ranking logic
│   │   ├── models/                       # Gamification database schemas
│   │   ├── rewards/                      # Reward system
│   │   ├── rules/                        # XP and level rules
│   │   └── wallet/                       # Coin balance and ledger
│   ├── pickup/
│   │   ├── pickup.model.ts               # Pickups and Device schemas
│   │   ├── pickup.routes.ts              # Pickup route definitions
│   │   ├── pickup.controller.ts          # Pickup request handlers
│   │   ├── pickup.service.ts             # Pickup business logic
│   │   ├── pickup.validation.ts          # Zod schemas & validate middleware
│   │   ├── pickup-state-machine.ts       # State transition validator
│   │   ├── rewards-client.ts             # HTTP client for external rewards
│   │   └── collection-center.*           # Collection center routes & logic
│   └── utils/
│       └── logger.ts                     # Structured console logger
├── package.json
└── tsconfig.json
```

---

## Getting Started

### Prerequisites

- **Node.js** v18+
- **MongoDB** running locally or a connection URI

### Installation

```bash
cd backend
npm install
```

### Environment Variables

Create a `.env` file in the `backend/` directory (you can use `.env.example` as a template):

```env
PORT=3000
MONGODB_URI=mongodb://localhost:27017/greencoin
JWT_SECRET=your_super_secret_jwt_key_here
REWARDS_SERVICE_URL=http://localhost:3001/api/v1/rewards/generate
```

### Running the Server

```bash
# Development (with ts-node)
npm run dev

# Build for production
npm run build
node dist/index.js

# Test (with Jest)
npm test
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js + TypeScript |
| Framework | Express 5 |
| Database | MongoDB via Mongoose |
| Validation | Zod |
| Auth | Real JWT middleware with bcrypt hashing |
| Testing | Jest, Supertest & mongodb-memory-server |
