# 🚋 trafic-radar (Traffic Delay & Operations Aggregation App)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

<p>
  <a href="README.en.md">English</a> |
  <a href="README.ja.md">日本語</a>
</p>

A web application that ingests public transit open data to aggregate delay and operation information and deliver notifications.
This is a modern Java application project by Kawaski and Nagai from Nohkai University, Department of Production Electronic Information Systems, 1st year.

---

## Table of Contents

1. Overview
2. Main Features
3. Feature Relationship Diagram
4. Tech Stack
5. Architecture Overview
6. Directory Structure
7. AI Usage & Development Policy
8. Sprint 0 Checklist (Getting Started)
9. Local Setup
10. Development Workflow
11. Test Strategy
12. Development Schedule
13. Future Plans
14. Team Members

---

## Overview

Public transit in Hokkaido (JR Hokkaido, local buses, etc.) often experiences delays and service suspensions due to weather and accidents. The information is scattered across operators, making it hard for users to get a real-time overall view.

This project periodically fetches and aggregates published operation data (GTFS / GTFS-Realtime, etc.) and provides a mechanism for users to view delays on a dashboard and receive notifications for routes they register.

The theme is motivated by interest in transportation infrastructure and aims to provide experience across a typical production workflow: data collection → asynchronous processing → notifications → visualization.

The project’s goals include learning modern server-side Java development, version control and collaboration with Git/GitHub, running infrastructure on AWS, and AI-assisted development using Gemini CLI.

---

## Main Features

Each feature explains its purpose, behavior, technologies used, and implementation notes. Refer to this section when dividing work or aligning expectations within the team.

### 1. Automated Operation Data Collection

Purpose
: Foundation for the app’s data. Periodically fetch static GTFS and dynamic GTFS-RT data and store/transform them into DB records for use by the dashboard and notifications.

Processing flow
:
1. GTFS (static) fetch: download ZIPs distributed by operators (low frequency, e.g., once per day)
   - Parse stops.txt, routes.txt, trips.txt, stop_times.txt
   - Update route master data in DB if content changed
2. GTFS-RT (dynamic) fetch: fetch real-time data in Protocol Buffers frequently (e.g., every 1–5 minutes)
   - Decode with gtfs-realtime-bindings
   - Extract statuses like "delay" or "service suspension"
3. Hand off to asynchronous processing: instead of saving directly to DB, push to SQS and separate parsing → notification decision → delivery (see feature 3)

Tech: Spring Scheduler / gtfs-realtime-bindings / Amazon SQS / Flyway

Implementation notes
:
- GTFS-RT is in Protocol Buffers; start by writing small code to decode and inspect contents
- Operator-provided GTFS formats and update frequencies vary—start with one data source for early validation
- JR Hokkaido may not publish structured GTFS; scraping might be required (investigate terms of use)

### 2. Route/Section Delay Dashboard

Purpose
: Present collected operation data in a user-friendly UI. This is the app’s face and the primary demo surface.

Processing flow
:
1. Frontend polls backend API for current delay status (TanStack Query)
2. Backend reads latest delay info from DB and returns it via REST
3. Frontend displays data in two modes:
   - Map view: overlay routes, stops, and delay status with MapLibre GL JS + react-map-gl
   - List view: table of route delays as an alternative for users who prefer lists

Tech: TanStack Query / MapLibre GL JS + react-map-gl / Tailwind CSS + shadcn/ui / openapi-typescript

Implementation notes
:
- Polling interval is a tradeoff between real-time feel and server load; start with 30s–60s
- Map implementation has higher learning cost—deliver a working list view first, then add the map
- Stop/route coordinates come from stops.txt; verify data before map work

### 3. Notifications for User-registered Routes

Purpose
: Allow users to register frequently used routes and receive proactive alerts when those routes experience delays. This feature provides hands-on experience with asynchronous design.

Processing flow
:
1. Detect delays from GTFS-RT (continuation of feature 1)
2. For detected delay events, determine recipients by:
   - Querying which users have registered this route
   - Checking notification conditions (e.g., only notify for delays longer than X minutes)
3. When targets are determined, send notifications via:
   - Email: Amazon SES
   - Chat: Slack / Charwork API

Incremental implementation plan
:
- Step 1 (learning): Use Spring Events (ApplicationEventPublisher) to experience in-process event-driven flow
- Step 2 (production-like): Introduce Amazon SQS to separate fetch → parse → notification decision → delivery

Tech: Spring Event / Amazon SQS / Amazon SES / Slack / Charwork API / WireMock (tests)

Implementation notes
:
- Design a user–route many-to-many (favorites) table so only registered routes trigger notifications
- Prevent duplicate notifications by tracking notification state for events
- Provide UI guidance for users to configure Slack/Charwork bot integration

### 4. Authentication & Authorization

Purpose
: Manage user identity and permissions (needed for favorites, admin functions, etc.).

Processing flow
:
1. User submits credentials via register/login forms
2. Backend authenticates and issues a JWT
3. Frontend stores the JWT and includes it with API requests
4. Backend verifies the JWT and authorizes each request

Roles and capabilities
:
- General user: can register/edit/delete own favorite routes
- Admin: manage data sources, notification rules, and other system settings

Tech: Spring Security + JWT

Implementation notes
:
- Enforce ownership checks in the Service layer, not just Controllers
- Decide on JWT expiry and refresh strategy early to avoid rework

### 5. Admin Functions

Purpose
: Manage system-level settings like data sources and notification rules (admin-only screens).

Processing flow
:
1. Admin logs in with elevated privileges
2. Admin uses data source UI to view/change bus companies and fetch intervals
3. Admin edits notification rules (e.g., delay threshold to trigger alerts)

Tech: Spring Security (role control) / Spring Data JPA

Implementation notes
:
- Lower priority than user-facing features; implement after features 1–4
- Start with configuration in a file and later expose UI for changes if time permits

---

## 🔗 Feature Relationship Diagram

```
[1. Automated Data Collection]
        │ detect delays
        ▼
[3. Notifications for Registered Routes] ── determine recipients ── [4. Auth & Authorization]
        │ reference stored data                              │ provide user/role info
        ▼                                                    ▼
[2. Route/Section Delay Dashboard]                    [5. Admin Functions]
         (viewable by all users)                        (admin-only settings)
```

Recommended implementation order: 4. Auth → 1. Data collection → 2. Dashboard → 3. Notifications → 5. Admin (Auth is foundation)

---

## 🛠 Tech Stack

### Backend

| Item | Tech |
|---|---|
| Language / Framework | Java 21 + Spring Boot 3.x |
| Build tool | Gradle |
| Web | Spring Web (REST API) |
| DB access | Spring Data JPA |
| Auth | Spring Security + JWT |
| Scheduling | Spring Scheduler |
| Migrations | Flyway |

### Asynchronous Processing

| Item | Tech |
|---|---|
| Learning step | Spring Event (ApplicationEventPublisher) |
| Production-like | Amazon SQS |

### Database

| Item | Tech |
|---|---|
| DB | PostgreSQL (consider PostGIS for geospatial queries) |
| Dev environment | Docker Compose |
| Production | Amazon RDS |

### External Data

| Item | Tech |
|---|---|
| GTFS (static) | Download and parse operator ZIPs |
| GTFS-RT (dynamic) | gtfs-realtime-bindings (Java Protocol Buffers) |
| JR Hokkaido | Official operation info (structured or require scraping) |

### Notifications

| Item | Tech |
|---|---|
| Email | Amazon SES |
| Chat | Slack / Charwork API |
| (Optional) | Firebase Cloud Messaging |

### Frontend

| Item | Tech |
|---|---|
| Framework | React + TypeScript |
| Build tool | Vite |
| Server state | TanStack Query (polling) |
| Client state | Zustand |
| Router | React Router |
| UI | Tailwind CSS + shadcn/ui |
| Map | MapLibre GL JS + react-map-gl |
| Type integration | openapi-typescript |

### Infrastructure (AWS)

| Item | Tech |
|---|---|
| API server | ECS (Fargate) |
| DB | RDS (PostgreSQL) |
| Queue | SQS |
| Email | SES |
| Static hosting | S3 + CloudFront |
| Scheduled tasks | EventBridge |
| IaC | AWS CDK (Java) |

### CI/CD & Testing

| Item | Tech |
|---|---|
| CI/CD | GitHub Actions |
| Unit tests | JUnit5 + Mockito |
| Integration tests | Testcontainers (PostgreSQL container) |
| External API mocks | WireMock |

---

## 🏗 Architecture Overview

```
[GTFS (static ZIP)]        [GTFS-RT (Protocol Buffers)]      [JR Hokkaido operation info]
      │                         │                              │
      └──────────────┬──────────┴──────────────────────────────┘
                      ▼
            [Fetcher (Spring Scheduler)]
                      │
                      ▼
             [SQS] ─ parse → decide → deliver (separated)
                      │
                      ▼
          [Spring Boot API Server] ── [RDS(PostgreSQL)]
                      │  REST API (OpenAPI)
                      ▼
        [React + TypeScript SPA (Vite)]
                      │
        ┌─────────────┴─────────────┐
        ▼                           ▼
 [MapLibre map view]         [TanStack Query polling]

Notification path: SQS → decision → SES / Slack / Charwork

Production: CloudFront → ALB → ECS(Fargate) → RDS
                              └→ S3 / SQS / EventBridge
```

---

## 📂 Directory Structure

### Current

```
trafic-radar/
├── .gitignore
├── GEMINI.md
└── README.md
```

### Planned

```
trafic-radar/
├── backend/               # Spring Boot app
│   ├── src/main/java/...
│   ├── src/main/resources/db/migration/  # Flyway migrations
│   └── src/test/java/...
├── frontend/              # React + TypeScript (Vite)
│   ├── src/
│   └── package.json
├── infra/                 # AWS CDK (Java)
├── docker-compose.yml
├── .github/workflows/
└── README.md
```

---

## 🤖 AI Usage & Development Policy

When using AI tools such as GitHub Copilot CLI or Gemini CLI, follow these rules:

### Principles
- Use AI as a helper; final decisions and responsibility remain with the development team.
- Clarify requirements, specs, and constraints before asking AI for help.
- Review AI suggestions for correctness, safety, and alignment with existing design before adoption.

### Rules
- Do not send secrets, API keys, or personal data to AI tools.
- Understand generated code before accepting it.
- Avoid large, design-changing edits that ignore original intent—prefer minimal, focused changes.
- Make changes reviewable via Git diffs.

### Quality & Safety
- Verify generated code with tests, builds, and static analysis.
- Confirm behavior matches specs after implementation.
- Consult the team if suggestions are unclear or risky.

### Expected Use
- Use AI for drafts, code completion, tests, and docs—but humans perform design and review

---

## 🏁 Sprint 0 Checklist (Getting Started)

Because it’s important to understand actual data shapes, GTFS exploration is prioritized before DB design.

### 1. Environment
- [ ] Install JDK 21
- [ ] Setup IntelliJ IDEA (or preferred IDE)
- [ ] Install Docker Desktop
- [ ] Install Node.js (LTS)
- [ ] Create GitHub repo (monorepo or separate repos; monorepo recommended for ease)

### 2. Touch GTFS data
- [ ] Download GTFS (static) from a local operator (e.g., Hokkaido Chuo Bus)
- [ ] Unzip and inspect CSVs (stops.txt, routes.txt, trips.txt, stop_times.txt)
- [ ] Fetch GTFS-RT once to experience Protocol Buffers (not human-readable)
- [ ] Note what to store in the DB

### 3. Spring Boot skeleton
- [ ] Generate project from Spring Initializr with dependencies: Spring Web, Spring Data JPA, PostgreSQL Driver, Flyway, Spring Security, Validation
- [ ] Run `./gradlew bootRun` locally to verify startup (app can be empty)

### 4. Docker Compose
- [ ] Create docker-compose.yml for PostgreSQL
- [ ] Verify Spring Boot connects to the Postgres container

### 5. React skeleton
- [ ] Create React+TypeScript app via `npm create vite@latest`
- [ ] Setup Tailwind CSS
- [ ] Verify a simple page renders

### 6. Product backlog
- [ ] Create a GitHub Project board
- [ ] Add issues as user stories based on the five main features

### 7. ER diagram
- [ ] Sketch tables: routes, stops, trips, delay_logs, users, favorites
- [ ] Create a diagram (draw.io or similar)

---

## 🚀 Local Setup

Prerequisites: Docker / Docker Compose, Java 21, Node.js (LTS)

Steps

```bash
# Clone
git clone https://github.com/<org>/trafic-radar.git
cd trafic-radar

# Start DB & services
docker compose up -d

# Backend (Flyway runs on startup)
cd backend
./gradlew bootRun

# Frontend (in another terminal)
cd frontend
npm install
npm run dev
```

---

## 👥 Development Workflow

- Task management: GitHub Issues + Projects
- Branching: main / develop / feature/* (simple Git Flow)
- Reviews: PR required; at least one reviewer approval before merge
- Sync: Weekly short meeting (15–30m)

---

## ✅ Test Strategy

- Unit tests: JUnit5 + Mockito for business logic
- Integration tests: Testcontainers with a real PostgreSQL container
- External API parts: WireMock for deterministic tests
- Run tests on push/PR via GitHub Actions



## 🔭 Future Plans

- Use PostGIS for advanced geospatial queries
- Add push notifications via Firebase Cloud Messaging
- Basic trend analysis (weekday/weather aggregation) and visualization



## References

- GTFS (General Transit Feed Specification): https://developers.google.com/transit/gtfs
- GTFS Realtime: https://developers.google.com/transit/gtfs-realtime
- gtfs-realtime-bindings (Java): https://github.com/MobilityData/gtfs-realtime-bindings
- Spring Boot: https://spring.io/projects/spring-boot
- MapLibre GL JS: https://maplibre.org
- OpenAPI: https://www.openapis.org
- AWS Docs (SQS/SES/RDS): https://docs.aws.amazon.com/

---

## License

MIT
