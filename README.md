# Krave

**Swipe-driven restaurant discovery powered by geospatial search, personalized ranking, and demand-driven catalog growth.**

[Live app](https://www.kravematch.com) · [Engineering blog](https://www.wmcvicar.me/blog/)

> This repository contains public documentation and architecture diagrams for Krave. The production source code is maintained in a private repository.

Krave helps people discover nearby restaurants through a swipe-based interface. Personal recommendations adapt to each user's behavior, while Group Discovery combines the learned preferences of selected friends without changing anyone's personal model.

The platform also expands geographically from real product demand. When someone opens Krave in an uncovered area, the backend records durable H3 demand and prioritizes that region for overnight ingestion.

---

## Features

- Location-based restaurant discovery using live GPS
- Swipe-driven Personal and Group Discovery
- Per-user preference models trained from swipe behavior
- Personalized restaurant ranking with logistic regression
- Taste Matches generated from predicted preference scores
- Friend system and combined group recommendations
- PostgreSQL/PostGIS geospatial queries with H3 indexing
- Demand-driven expansion into previously uncovered areas
- Automated overnight restaurant ingestion and catalog refreshes
- Privacy-conscious product analytics with PostHog

---

## Architecture

### Krave system architecture

The online request path serves discovery, social features, and recommendation scoring. Catalog expansion runs asynchronously on EC2 and writes into the shared PostgreSQL/PostGIS datastore.

![Krave system architecture](./high_level_architecture.png)

### Personal + group recommendation architecture

Personal and Group Discovery use the same nearby candidate layer while maintaining separate interaction state. Group swipes do not retrain personal models or consume the user's Personal Discovery queue.

![Personal and group recommendation architecture](./recommendation_architecture.png)

### Demand-driven restaurant ingestion

A stack request can record durable H3 demand without waiting for ingestion. Prioritized regions are processed asynchronously and become available in later Discover stacks.

![Demand-driven restaurant ingestion](./demand_driven_ingestion.png)

---

## Recommendation system

### Personal Discovery

Each personal swipe stores the feature snapshot used at prediction time. Once an account has enough balanced swipe history, Krave trains an L2-regularized logistic-regression model for that user.

Nearby restaurants are transformed into feature vectors and assigned a predicted preference probability. Krave uses this score to rank the Personal Discovery stack and identify qualified Taste Matches.

The model is retrained periodically from recent personal interactions so recommendations can adapt as the user's preferences change.

### Group Discovery

Group Discovery evaluates nearby restaurants using the existing taste models of selected friends.

The same candidate restaurants and extracted features are shared across both recommendation paths, but group interaction state remains independent:

- Group swipes do not retrain personal models
- Group swipes do not remove restaurants from personal queues
- Each member retains an independent preference model
- Group scoring considers both overall agreement and weaker individual preferences
- Qualified restaurants can become Group Picks

---

## Demand-driven coverage

A missing restaurant stack can represent several different states:

- the area has no current catalog coverage
- the user has already viewed the eligible restaurants nearby
- active filters removed every candidate
- the area contains fewer restaurants than the requested stack size

Krave classifies these states separately rather than treating every empty stack as the same problem.

When an area is truly uncovered, the API records demand across nearby H3 cells. Distinct users create a stronger scheduling signal than repeated refreshes from one account.

The overnight ingestion pipeline then:

1. Selects an eligible H3 tile by priority
2. Leases the tile using database locking
3. Runs the ingestion worker
4. Validates and normalizes restaurant records
5. Performs stable CID-based inserts or updates
6. Assigns PostGIS geography and H3 indexing
7. Records the result and releases the lease

New ingestion work may begin between **2:00 AM and 7:00 AM** in the `America/Toronto` timezone. Work started before the cutoff may finish afterward, but no additional tile begins once the window closes.

---

## Geospatial search

Krave combines two spatial systems:

### H3

H3 divides geographic space into hexagonal cells. Krave uses these cells to:

- organize restaurant coverage
- record geographic demand
- prioritize ingestion work
- group nearby catalog regions
- reduce the search space before precise distance calculations

### PostGIS

PostGIS remains the source of truth for exact geographic filtering. After H3 narrows the candidate region, PostGIS calculates precise distances and enforces the user's selected search radius.

This combination keeps location queries efficient without sacrificing distance accuracy.

---

## Restaurant ingestion

Incoming restaurant records pass through an incremental ingestion pipeline rather than replacing the complete catalog.

The pipeline:

- validates required fields
- normalizes metadata, tags, photos, and reviews
- detects existing restaurants using stable identifiers
- enriches changed records
- inserts newly discovered restaurants
- assigns PostGIS and H3 geographic data
- records ingestion results for monitoring and retries

Overlapping geographic jobs may return the same restaurant, so CID-based upserts prevent duplicate catalog rows.

---

## Analytics and observability

Krave uses PostHog to understand product behavior and performance, including:

- Discover and stack activity
- Personal and Group swipes
- Taste Matches and Group Picks
- uncovered-area demand
- page and stack latency
- friend and group feature adoption

Analytics events are filtered to avoid sending exact GPS coordinates, H3 identifiers, authentication tokens, or raw private user data.

Backend services and ingestion workers are monitored separately through system logs and persisted scrape results.

---

## Technology

| Layer | Technologies |
|---|---|
| Frontend | React, TypeScript, Vite |
| Backend | Python, FastAPI |
| Ingestion | Go, Python orchestration |
| Database | PostgreSQL, PostGIS |
| Spatial indexing | H3 |
| Recommendations | Logistic Regression |
| Authentication | Google Identity Services |
| Analytics | PostHog |
| Frontend deployment | Vercel |
| Backend deployment | AWS EC2, systemd |

---

## Engineering write-ups

More detailed explanations of Krave's systems are available on my portfolio:

- [How H3 Spatial Indexing Works and How I Implemented It into Krave](https://www.wmcvicar.me/blog/how-h3-spatial-indexing-works/)
- [Designing a Scalable Restaurant Ingestion Pipeline](https://www.wmcvicar.me/blog/designing-a-scalable-restaurant-ingestion-pipeline/)

---

## Project status

Krave is under active development.

Current areas of focus include:

- improving recommendation calibration
- expanding geographic coverage
- reducing stack and page latency
- refining Group Discovery
- strengthening ingestion reliability and observability

---

## Source availability

Krave's production source code, deployment configuration, and database migrations are maintained in a private repository.

This public repository is used to share:

- system architecture
- technical documentation
- engineering diagrams
- public project updates
