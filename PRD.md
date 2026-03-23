# Product Requirements Document: AdHoc Delivery System

## 1. Overview

**Product Name:** AdHoc
**One-liner:** On-demand delivery system that moves anything from Point A to Point B.
**Author:** Ari Choudhary
**Date:** 2026-03-23
**Status:** Draft

AdHoc is a lightweight, real-time delivery platform where a **sender** posts a delivery request (pickup location, drop-off location, package details) and a nearby **courier** claims and fulfills it. Think of it as "Uber for ad-hoc deliveries" — no scheduled routes, no fleet management, just point-to-point on-demand transport.

---

## 2. Problem Statement

Moving items between two locations today requires either:
- **Traditional couriers** — slow, expensive, require scheduling in advance.
- **Asking a friend** — unreliable, doesn't scale.
- **Ride-share workarounds** — against TOS, no package tracking, no accountability.

There is no simple, fast, affordable way to say: *"I need this thing moved from here to there, right now."*

---

## 3. Target Users

| Persona | Description | Key Need |
|---------|-------------|----------|
| **Sender** | Anyone who needs something delivered — individuals, small businesses, event organizers | Post a delivery, track it in real time, pay on completion |
| **Courier** | Gig workers, bike messengers, drivers looking for flexible income | See nearby requests, claim jobs, get paid quickly |
| **Admin** (v2) | Platform operator | Monitor deliveries, handle disputes, view analytics |

---

## 4. Core User Flows

### 4.1 Sender Flow
1. Open app / web dashboard
2. Enter **pickup address** (Point A) and **drop-off address** (Point B)
3. Describe the package (size category: envelope / small / medium / large)
4. See **estimated price** and **ETA**
5. Confirm and post the delivery request
6. Get matched with a courier
7. Track courier in real time on a map
8. Receive confirmation when delivered (photo proof)
9. Rate the courier

### 4.2 Courier Flow
1. Go online / toggle availability
2. See a feed of nearby delivery requests with pickup distance, drop-off distance, and payout
3. Claim a delivery
4. Navigate to pickup (Point A) — confirm pickup with a photo or code
5. Navigate to drop-off (Point B) — confirm delivery with a photo
6. Get paid (earnings added to wallet)
7. Rate the sender

### 4.3 Matching Logic
- When a sender posts a request, notify couriers within a configurable radius (default: 5 km of Point A)
- First courier to claim gets the job (FCFS)
- If unclaimed for 3 minutes, expand radius by 2 km and re-notify
- If unclaimed for 10 minutes, notify sender and offer to cancel or keep waiting

---

## 5. Features — MVP (v1)

### 5.1 Delivery Management
| Feature | Description | Priority |
|---------|-------------|----------|
| Create delivery request | Sender enters A, B, package size, optional notes | P0 |
| Price estimation | Distance-based pricing with size multiplier | P0 |
| Delivery feed | Couriers see available requests sorted by proximity | P0 |
| Claim delivery | Courier accepts a delivery request | P0 |
| Real-time tracking | Live map showing courier location between A and B | P0 |
| Delivery confirmation | Photo proof at pickup and drop-off | P0 |
| Delivery history | Past deliveries for both sender and courier | P1 |

### 5.2 User Accounts
| Feature | Description | Priority |
|---------|-------------|----------|
| Sign up / login | Email + password, or OAuth (Google/Apple) | P0 |
| User profiles | Name, photo, phone number, vehicle type (courier) | P0 |
| Ratings | 5-star mutual rating after each delivery | P1 |

### 5.3 Payments
| Feature | Description | Priority |
|---------|-------------|----------|
| Price calculation | Base fee + per-km rate + size multiplier | P0 |
| Payment capture | Charge sender on delivery confirmation (Stripe) | P0 |
| Courier payouts | Weekly payout to courier's bank account | P1 |
| Tipping | Optional tip after delivery | P2 |

### 5.4 Notifications
| Feature | Description | Priority |
|---------|-------------|----------|
| Push notifications | New request nearby (courier), courier assigned (sender), delivery complete | P0 |
| SMS fallback | For critical updates (delivery confirmed) | P1 |

---

## 6. Features — Post-MVP (v2+)

- **Scheduled deliveries** — book a delivery for a future time
- **Multi-stop routes** — A -> B -> C
- **Admin dashboard** — delivery analytics, dispute resolution, courier management
- **In-app chat** — sender <-> courier messaging
- **Insurance / liability** — optional package insurance
- **Delivery batching** — combine nearby deliveries for couriers going the same direction
- **API for businesses** — programmatic delivery creation for e-commerce, restaurants, etc.

---

## 7. Technical Architecture

### 7.1 Stack

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Mobile App** (`app/`) | React Native (Expo) | Cross-platform iOS + Android from one codebase |
| **Web Frontend** (`frontend/`) | Next.js 14 (App Router) | Sender dashboard, courier onboarding, admin (v2) |
| **Backend API** (`backend/`) | Node.js + Express | REST API + WebSocket for real-time tracking |
| **Database** | PostgreSQL + PostGIS | Geospatial queries for proximity matching |
| **Real-time** | Socket.IO | Live courier location updates |
| **Maps** | Mapbox or Google Maps API | Geocoding, routing, ETA, live map |
| **Payments** | Stripe Connect | Sender charges + courier payouts |
| **Auth** | Supabase Auth or Auth0 | JWT-based, OAuth providers |
| **File Storage** | Supabase Storage or S3 | Delivery proof photos |
| **Infra** (`infra/`) | Docker + Fly.io | Containerized deploy, geo-distributed |

### 7.2 Data Model (Core Entities)

```
User
  id, email, name, phone, role (sender|courier), avatar_url,
  vehicle_type (courier only), rating_avg, created_at

Delivery
  id, sender_id, courier_id (nullable until claimed),
  status (pending|claimed|picked_up|delivered|cancelled),
  pickup_address, pickup_lat, pickup_lng,
  dropoff_address, dropoff_lat, dropoff_lng,
  package_size (envelope|small|medium|large),
  notes, price_cents, distance_km, eta_minutes,
  pickup_photo_url, dropoff_photo_url,
  created_at, claimed_at, picked_up_at, delivered_at

Rating
  id, delivery_id, from_user_id, to_user_id, score (1-5), comment

Payment
  id, delivery_id, stripe_payment_intent_id, amount_cents,
  status (pending|captured|refunded), tip_cents
```

### 7.3 Key API Endpoints

```
POST   /api/auth/signup
POST   /api/auth/login

POST   /api/deliveries              # create request (sender)
GET    /api/deliveries/available     # nearby feed (courier), query: lat, lng, radius
POST   /api/deliveries/:id/claim    # claim (courier)
POST   /api/deliveries/:id/pickup   # confirm pickup + photo
POST   /api/deliveries/:id/deliver  # confirm delivery + photo
GET    /api/deliveries/:id          # delivery details + tracking
GET    /api/deliveries/history       # user's past deliveries

POST   /api/deliveries/:id/rate     # rate the other party
GET    /api/users/me                # profile
PUT    /api/users/me                # update profile

WS     /ws/tracking/:delivery_id    # real-time courier location
```

### 7.4 Pricing Formula

```
price = BASE_FEE + (distance_km * PER_KM_RATE * SIZE_MULTIPLIER)

BASE_FEE       = $3.00
PER_KM_RATE    = $1.50
SIZE_MULTIPLIER:
  envelope = 1.0
  small    = 1.0
  medium   = 1.3
  large    = 1.6
```

Platform takes 20% commission; courier receives 80%.

---

## 8. Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **API latency** | p95 < 200ms for non-geo queries, < 500ms for geo queries |
| **Real-time tracking** | Location updates every 3 seconds |
| **Availability** | 99.9% uptime |
| **Security** | HTTPS everywhere, JWT auth, input validation, rate limiting |
| **Privacy** | Courier location only shared during active delivery, GDPR-ready |
| **Scalability** | Support 10k concurrent deliveries in v1 |

---

## 9. Success Metrics

| Metric | Target (3 months post-launch) |
|--------|-------------------------------|
| Deliveries completed per day | 100+ |
| Average delivery time (post-claim) | < 45 min for < 10 km |
| Courier claim rate | > 80% of requests claimed within 5 min |
| Sender satisfaction (avg rating) | > 4.5 / 5 |
| Courier retention (monthly active) | > 60% |

---

## 10. Milestones

| Phase | Scope | Timeline |
|-------|-------|----------|
| **Phase 1 — Backend Core** | Auth, delivery CRUD, geo matching, pricing engine | Week 1-2 |
| **Phase 2 — Real-time** | WebSocket tracking, push notifications | Week 2-3 |
| **Phase 3 — Web Frontend** | Sender dashboard (create, track, history) | Week 3-4 |
| **Phase 4 — Mobile App** | Courier app (feed, claim, navigate, confirm) | Week 4-6 |
| **Phase 5 — Payments** | Stripe integration, sender charges, courier payouts | Week 5-6 |
| **Phase 6 — Polish & Launch** | Testing, photo upload, ratings, deploy | Week 6-8 |

---

## 11. Open Questions

- [ ] Which maps provider? Mapbox (cheaper) vs Google Maps (better known)?
- [ ] Courier verification — require ID upload before going live?
- [ ] Geo-fence to specific cities at launch, or open everywhere?
- [ ] What is the maximum package weight/size we support?
- [ ] Do we need a web experience for couriers, or mobile-only?
