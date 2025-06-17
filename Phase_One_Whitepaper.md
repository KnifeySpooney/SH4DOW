# SH4DOW: Secure Honeypot for Digital Operational Warfare
## Whitepaper: Phase 1 — The Sentinel's Dawn

### Technical Overview (Expanded)

## 1. The Listener Layer — Go-Based Network Daemons

Each SH4DOW listener is written in Golang and compiled as a static binary. These daemons expose emulated services (e.g., Telnet, FTP, HTTP, SSH) on configurable ports with minimal system overhead. Internally, the architecture leverages goroutines and channel-based IO to asynchronously handle inbound traffic, ensuring near-zero blocking and scalable throughput under horizontal deployment.

Listeners do not attempt full protocol emulation. Instead, they focus on mimicking enough handshake and banner behavior to convince low- and mid-tier adversaries or bots that they're interfacing with a real host. Configurable response templates allow variation in banners, timing behavior, and supported commands. Session state is logged as JSON-formatted event objects. TCP fingerprinting and packet-level anomaly checks are included optionally, using a lightweight inspection module built with gopacket or equivalent.

Each listener includes a pluggable filter module for pre-processing connections, useful for dropping known benign probes, internal traffic, or CDN-originated noise.

## 2. The Core Engine — Python + FastAPI Intelligence Layer

The core receives event payloads from listeners via REST (or optionally via gRPC in future versions), each containing metadata like source IP, port, timestamp, raw payload, and session context. FastAPI exposes these as async endpoints, backed by an event loop that queues incoming data for batch processing.

Event objects are then routed through a pipeline consisting of:

- **Pre-parser:** Normalizes field names, enforces schema conformity, assigns UUIDs
- **Parser:** Converts raw payloads into structured records; optional regex modules tag known exploit signatures (e.g., Mirai variants)
- **Profiler Stub:** Adds stub metadata to build per-IP profile templates — OS guess, frequency of visits, observed behaviors
- **Dispatch Layer:**
  - Log archive sent to Elasticsearch
  - Profile data stored in PostgreSQL
  - Internal WebSocket event bus emits to F0RT frontend and SP3CTR engine

The engine includes a modular threat tagging subsystem. These tags are key-value pairs (e.g., `scan_type=horizontal`, `tool=mimikatz_lure`, `stage=initial_probe`) applied by pattern matchers or stateful rules, with all tags stored relationally for search and timeline use.

Authentication is token-based with time-rotated secrets, and all API traffic is intended to be deployed behind an NGINX reverse proxy with WAF or rate-limiting middleware.

## 3. The Database Stack — Hybrid Index + Relational Memory

### 3.1 Elasticsearch Cluster:

Deployed as a single-node or multi-node instance, depending on footprint. Optimized for time-based indices with rollover strategies (daily or hourly depending on traffic volume). Key indices:

- **events-*:** Raw connection and session logs
- **flags-*:** Tag history and incident labels
- **streams-*:** Aggregated behavior timelines

Data ingested via bulk API with parsing handled in Core. Kibana-compatible for direct queries, but Phase 1 favors backend-driven queries exposed through the Core API for unified access control.

### 3.2 PostgreSQL:

Serves as structured state memory. Tables include:

- **actor_profiles:** IP, ASN, first_seen, last_seen, observed tags
- **service_templates:** Which services were exposed during a given session
- **system_state:** Configuration snapshots, listener status, frontend controls

Intended to eventually track multi-session behaviors across time or geography. Phase 1 schema includes foreign key enforcement and basic normalization (e.g., tags as lookup tables).

## 4. The UI Console — React-Based F0RT Module

The frontend module for SH4DOW lives within the F0RT dashboard and is built with React and Tailwind CSS, following a component-driven design. Pages are lazy-loaded and structured as follows:

### Live Map Panel:
Uses Leaflet.js or Mapbox to render attacker origin points, enriched via GeoIP lookups. Points animate on ingress, with optional clustering and zoom-to-event features.

### Timeline View:
A scrollable, filterable sequence of recorded events, sorted by time or severity tag. Custom components highlight major attacker behaviors or escalation attempts. Color-coded badges map to tags provided by the Core Engine.

### Session Replay:
Modal component that reconstructs the flow of a single connection or session using raw data from Elasticsearch. Shows handshake timing, commands issued, and server responses. Future expansion will include playback of longer interactions as pseudo-terminal scrolls.

### Profiler Panel:
View per-IP actor cards with metadata drawn from PostgreSQL — frequency, preferred ports, recurring behavior, linked tool signatures. Cards include activity graphs and timeline plots.

The UI queries the Core Engine's API via bearer-auth headers and supports dark/light mode via CSS variables. Alert banners and live stream feeds are handled by a local WebSocket daemon proxying events from the backend bus.

## 5. System Design: Deployment & Communication

Each SH4DOW instance is modular. A full deployment stack includes:

- Listener containers (Go binaries wrapped in Docker)
- Core Engine (Python FastAPI app, possibly in Uvicorn/Gunicorn behind NGINX)
- Elasticsearch container
- PostgreSQL container
- React frontend, optionally reverse-proxied by same NGINX instance

### Inter-service comms:

- **Listener → Core:** REST (Phase 1), gRPC planned for Phase 2
- **Core → DBs:** Native drivers (Elasticsearch DSL + psycopg2/asyncpg)
- **Core → SP3CTR:** REST/WS with optional integration stub for correlation
- **Core → F0RT frontend:** REST + WebSocket pub/sub

Each module supports containerization for orchestration under Docker Compose or Kubernetes. Logs are centralized via a logging agent (e.g., Fluent Bit) for optional external forwarding in enterprise deployments.
