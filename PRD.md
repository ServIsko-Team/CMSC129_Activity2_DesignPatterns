# IskoLuv — Product Requirements Document (PRD)

**Document Version:** 1.0  
**Status:** Draft  
**Course:** CMSC 129 Software Engineering II — Activity #2  
**Institution:** University of the Philippines Visayas  
**Semester:** 2nd Semester AY 2025-2026  

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [Goals and Objectives](#2-goals-and-objectives)
3. [Target Users](#3-target-users)
4. [Core Features](#4-core-features)
5. [System Architecture](#5-system-architecture)
6. [Design Pattern Decisions](#6-design-pattern-decisions)
7. [Non-Functional Requirements](#7-non-functional-requirements)
8. [Out of Scope](#8-out-of-scope)
9. [Risks and Mitigations](#9-risks-and-mitigations)
10. [Glossary](#10-glossary)

---

## 1. Product Overview

### 1.1 Product Name
**IskoLuv**

### 1.2 Tagline
*"Di ka makahanap ng match sa bahay. Lakad ka."*

### 1.3 Summary

IskoLuv is a proximity-based dating and social discovery mobile application designed exclusively for the **University of the Philippines Visayas (UPV) Miagao** campus community. The application gamifies campus exploration by requiring users to physically be near one another before a profile becomes discoverable — creating organic, in-person encounters that lead to genuine connections.

### 1.4 The Twist: *Catch 'Em All* — Encounter Mechanic

IskoLuv's core differentiator from existing dating platforms is its **physical-presence unlock system**. Profiles are hidden by default. A user's profile is only revealed and made swipeable when:

- Two users come **within a 10-meter radius** of each other on campus, OR
- A user visits a designated **campus landmark** (e.g., the Oblation, the DAC, the Fish Port)

This mechanic transforms the app from a passive swipe-and-match tool into an active **campus scavenger hunt**, incentivizing students to explore UPV Miagao and meet their peers face-to-face before a digital connection is made.

---

## 2. Goals and Objectives

### 2.1 Product Goals

| # | Goal | Success Metric |
|---|------|----------------|
| G1 | Encourage physical campus exploration | Average user visits ≥ 3 distinct campus zones per session |
| G2 | Foster in-person first interactions | ≥ 60% of matches preceded by physical proximity event |
| G3 | Provide low-latency encounter detection | Proximity event detected and pushed in < 500ms |
| G4 | Maintain a lightweight, battery-efficient client | Client-side battery drain ≤ standard social media apps |
| G5 | Deliver a clean and maintainable codebase | No ViewModel exceeds single-responsibility; all patterns documented |

### 2.2 Academic Objectives (CMSC 129)

- Demonstrate correct application of **one Creational**, **one Structural**, and **one Behavioral** design pattern.
- Justify each design pattern choice relative to a concrete app feature.
- Produce clear visual diagrams (Mermaid) illustrating With vs. Without pattern comparisons.
- Write accurate pseudocode for each pattern implementation.

---

## 3. Target Users

### 3.1 Primary Users

**Enrolled UPV Miagao Students**
- Age range: 18–25
- On-campus daily or several times per week
- Comfortable with mobile apps; familiar with Pokémon GO-style mechanics
- Looking for social connection within their campus community

### 3.2 User Needs

| User Need | IskoLuv Solution |
|-----------|-----------------|
| Meet people in a low-pressure way | Physical proximity breaks the ice before any digital interaction |
| Discovery tied to campus life | Landmark-based unlocks reward campus engagement |
| Real-time, responsive experience | WebSocket push via Observer pattern — no polling delay |
| Consistent data across app screens | Singleton IskodexStore ensures unified state everywhere |
| Simple, clutter-free UI | Facade hides backend complexity from presentation layer |

---

## 4. Core Features

### 4.1 Proximity-Based Profile Discovery

**Description:** User profiles are locked by default. They are unlocked only when the app backend detects that two users are within 10 meters of each other via Redis GEOSEARCH.

**Behavior:**
- User location is streamed to the Go/Nakama backend via WebSocket
- Backend performs Redis `GEOSEARCH` at regular intervals against active user coordinates
- Upon detecting proximity (≤ 10m), the `ProximityManager` pushes an encounter event to both users' connected streams
- The Flutter client receives the push, triggers haptic feedback, and opens an encounter dialog

**Design Pattern Used:** Observer (ProximityManager as Subject; Flutter WebSocket clients as Observers)

---

### 4.2 The Iskodex (Caught Profile Gallery)

**Description:** A Pokédex-style gallery of all profiles the user has "caught" through campus encounters. Profiles appear greyed out until unlocked, and unlock on encounter.

**Behavior:**
- All caught/unlocked profiles are stored in the `IskodexStore`
- The `IskodexStore` is a Singleton — one instance is shared across the MapView, IskodexView, and notification handlers
- State updates to the store reactively propagate to all bound ViewModels

**Design Pattern Used:** Singleton (IskodexStore)

---

### 4.3 Encounter Capture Flow

**Description:** When a proximity event fires, the app executes a multi-step capture sequence: local state is updated, haptic feedback fires, and the backend is notified via RPC.

**Behavior:**
- `MapViewModel` calls `DiscoveryFacade.handleCapture(profile)`
- The Facade coordinates: (1) saving profile to `IskodexStore`, (2) triggering `HapticEngine`, (3) sending RPC to `NakamaClient`
- ViewModel remains decoupled from all subsystem implementation details

**Design Pattern Used:** Facade (DiscoveryFacade)

---

### 4.4 Campus Landmark Zones

**Description:** Designated GPS-geofenced zones around key campus landmarks. Visiting a landmark zone unlocks a batch of nearby profiles even without peer-to-peer proximity.

**Landmarks (Initial Set):**
- The Oblation
- DAC (Division of Arts and Communication Building)
- The Fish Port

---

### 4.5 User Profile & Archetype System

**Description:** Each user selects an "archetype" (e.g., The Org Leader, The Thesis Grinder, The Tambay sa DAC) that appears on their profile card. Archetypes are cosmetic and used for display in the Iskodex.

---

## 5. System Architecture

### 5.1 Architecture Overview

IskoLuv follows a **client-server architecture** with a Flutter MVVM frontend and a Go/Nakama game-server backend.

```
┌─────────────────────────────────────────┐
│            Flutter Client               │
│  ┌─────────┐  ┌──────────────────────┐  │
│  │  Views  │◄─│     ViewModels       │  │
│  └─────────┘  └──────┬───────────────┘  │
│                      │                  │
│              ┌───────▼──────────┐       │
│              │DiscoveryFacade   │       │  ◄── Facade Pattern
│              └───┬──────────────┘       │
│                  │                      │
│  ┌───────────────▼──────────────────┐   │
│  │         IskodexStore (Singleton) │   │  ◄── Singleton Pattern
│  └──────────────────────────────────┘   │
└──────────────────┬──────────────────────┘
                   │ WSS / RPC
┌──────────────────▼──────────────────────┐
│           Nakama Go Runtime             │
│  ┌──────────────────────────────────┐   │
│  │       ProximityManager           │   │  ◄── Observer Pattern
│  │  (Observer Subject)              │   │
│  └───────────┬──────────────────────┘   │
│              │                          │
│  ┌───────────▼──────────────────────┐   │
│  │    Redis (Hot Spatial Data)      │   │
│  │    PostGIS (Cold Permanent Data) │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### 5.2 Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Mobile Client | Flutter | Cross-platform iOS/Android UI |
| State Architecture | MVVM | Separation of View, ViewModel, Model |
| Game Backend | Go on Nakama | Real-time multiplayer server runtime |
| Hot Spatial Cache | Redis (GEOSEARCH) | Sub-millisecond proximity queries |
| Persistent Storage | PostGIS (PostgreSQL) | Permanent geospatial and user records |
| Real-time Transport | WebSockets (WSS) | Bidirectional client-server streaming |
| Remote Procedure Calls | Nakama RPC | Triggered server-side actions (e.g., capture confirmation) |

---

## 6. Design Pattern Decisions

### 6.1 Pattern Selection Summary

| Pattern Type | Pattern Chosen | Applied To | Justification |
|---|---|---|---|
| Creational | **Singleton** | IskodexStore | One globally consistent state store; prevents data fragmentation across ViewModels |
| Structural | **Facade** | DiscoveryFacade | Encapsulates multi-subsystem encounter flow behind a single ViewModel-facing API |
| Behavioral | **Observer** | ProximityManager | Eliminates polling; enables event-driven, server-push proximity notifications |

---

### 6.2 Pattern 1: Singleton — IskodexStore

**Problem Solved:** In a multi-screen Flutter app following MVVM, multiple ViewModels (MapViewModel, IskodexViewModel) both require access to the list of caught profiles. Without a Singleton, each ViewModel could instantiate a separate copy of the store, leading to data desynchronization (a "ghost match" appearing on one screen but not another).

**Solution:** The `IskodexStore` is implemented as a Dart Singleton. Its factory constructor always returns the same private static instance, ensuring all ViewModels share one source of truth. Any state change — such as a new profile being caught — is immediately reflected across all bound UI components.

**Trade-offs Accepted:**
- Global state can complicate unit testing (mitigated via dependency injection in tests)
- Requires `notifyListeners()` discipline to avoid stale UI

---

### 6.3 Pattern 2: Facade — DiscoveryFacade

**Problem Solved:** The capture flow involves three distinct subsystems: the `NakamaClient` (network), `IskodexStore` (local state), and `HapticEngine` (device hardware). Without a Facade, the `MapViewModel` would need to import, manage, and sequence all three — violating the single-responsibility principle and making future changes to any subsystem ripple into the presentation layer.

**Solution:** `DiscoveryFacade` provides one method — `handleCapture(Profile)` — that internally sequences all three subsystem calls. The ViewModel only depends on the Facade interface, not on any underlying service.

**Trade-offs Accepted:**
- Facade can become a "God object" if too many responsibilities are added over time (mitigated by strict scope: capture flow only)

---

### 6.4 Pattern 3: Observer — ProximityManager

**Problem Solved:** Proximity must be detected in real time. A naive implementation would have the Flutter client poll the server repeatedly ("Are we close? How about now?"), causing excessive battery drain, data consumption, and server load — especially with many concurrent users.

**Solution:** The Go-based `ProximityManager` acts as the Subject. Every connected Flutter client is registered as an Observer via WebSocket stream. The manager runs `GEOSEARCH` queries against Redis and only emits an event when two users cross the 10-meter threshold — keeping the system idle otherwise.

**Trade-offs Accepted:**
- Persistent WebSocket connections consume server resources; mitigated by Nakama's efficient connection pooling
- Subscriber management (subscribe/unsubscribe on app lifecycle) must be carefully handled to prevent memory leaks

---

## 7. Non-Functional Requirements

### 7.1 Performance

| Requirement | Target |
|-------------|--------|
| Proximity event detection latency | < 500ms end-to-end |
| App cold start time | < 3 seconds |
| Redis GEOSEARCH query time | < 10ms |
| WebSocket message delivery | < 100ms under normal campus network |

### 7.2 Reliability

- The `IskodexStore` must persist caught profiles locally (device storage) to survive app restarts.
- WebSocket reconnection logic must handle intermittent campus Wi-Fi drops gracefully.

### 7.3 Privacy & Safety

- Location data is processed server-side and never exposed raw to other users.
- Only a binary "encounter event" is sent to client — exact coordinates of other users are never transmitted to peers.
- Users can opt out of discoverability at any time (go "invisible").

### 7.4 Scalability

- Redis GEOSEARCH is designed to handle thousands of concurrent user coordinates efficiently.
- Nakama's Go runtime supports horizontal scaling for peak usage periods (e.g., enrollment week, intramurals).

### 7.5 Usability

- Core encounter mechanic must be understandable within the first 60 seconds of use (onboarding tutorial required).
- Haptic + visual feedback on encounter must feel immediate and satisfying.

---

## 8. Out of Scope

The following features are explicitly excluded from the initial version:

- Chat/messaging between matched users (planned for v2)
- Cross-campus discovery (other UPV campuses)
- Profile photo uploads (text-and-archetype only in v1 for privacy)
- Paid premium features or subscriptions
- Desktop or web client

---

## 9. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| GPS inaccuracy on campus (buildings, signal interference) | High | High | Use Redis GEOSEARCH tolerance tuning; allow ±2m buffer |
| Low initial user adoption (cold-start problem) | Medium | High | Launch during high-foot-traffic campus events; enable landmark zone unlocks to provide value even without peers nearby |
| Privacy concerns around location tracking | Medium | High | Never expose raw coordinates to peers; strict server-side processing only |
| WebSocket connection instability on campus Wi-Fi | Medium | Medium | Implement exponential backoff reconnection; local queue for missed events |
| Singleton becoming a bottleneck for testing | Low | Medium | Provide test-injectable factory override for unit tests |

---

## 10. Glossary

| Term | Definition |
|------|-----------|
| **Iskodex** | The in-app gallery of all profiles a user has "caught" — a portmanteau of "Isko/Iska" and "Pokédex" |
| **Encounter** | The event triggered when two users come within 10 meters of each other |
| **Capture** | The act of unlocking another user's profile following an encounter |
| **Archetype** | A cosmetic persona tag assigned to each user profile (e.g., "The Thesis Grinder") |
| **Subject (Observer Pattern)** | The object that maintains a list of observers and notifies them of state changes |
| **Observer (Observer Pattern)** | An object that registers with a Subject to receive event notifications |
| **Singleton** | A creational design pattern that restricts a class to a single instance |
| **Facade** | A structural design pattern that provides a simplified interface over complex subsystems |
| **MVVM** | Model-View-ViewModel; a UI architecture pattern separating data, business logic, and presentation |
| **RPC** | Remote Procedure Call; a mechanism to invoke server-side functions from the client |
| **PostGIS** | A PostgreSQL extension that adds support for geographic objects and spatial queries |
| **GEOSEARCH** | A Redis command that queries members within a given radius of a coordinate |
| **WSS** | WebSocket Secure; the encrypted WebSocket protocol used for real-time bidirectional communication |
| **Nakama** | An open-source game server framework used as IskoLuv's backend runtime |

---

*IskoLuv PRD v1.0 — CMSC 129 Activity #2*  
*University of the Philippines Visayas — 2nd Semester AY 2025-2026*
