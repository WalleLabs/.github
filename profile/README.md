# WalleLabs

**Every place gets its own AI agent.** Talk to a store's agent when you're standing in it, or let your personal agent
find the nearest one and order for you, agent to agent. Working name: **Nyby**.

Shops, cafes, buildings, parks and museums each have an agent configured by its owner. The agents speak open
protocols (A2A for agent-to-agent messaging), so any compatible agent can talk to them, not just ours.

> Status: early build. Discovery, geofencing and the first mobile app and website run end to end on a local stack.
> The backend is now one Node + TypeScript service; store agents and the personal agent are being ported to it.

## How it fits together

```mermaid
flowchart LR
    App["📱 Mobile app<br/>(Expo)"]
    Web["🌐 Website<br/>(Next.js :3000)"]
    Orch["🤖 Orchestrator<br/>your personal agent<br/>(being ported)"]
    Reg["📍 Registry<br/>discovery · geofencing :8787"]
    Store["🏪 Store agents<br/>A2A · menu · cart · orders<br/>(being ported)"]
    DB[("🗄️ Postgres + PostGIS")]
    LLM["🧠 LLM<br/>Anthropic"]

    Web -->|"live demo"| Reg
    App -->|"chat from anywhere"| Orch
    App -->|"which agents are near me?"| Reg
    App -->|"chat in the store"| Store
    Orch -->|"find places"| Reg
    Orch -->|"talk / order (A2A)"| Store
    Orch --> LLM
    Store --> LLM
    Reg --> DB
    Store --> DB
    Orch --> DB
```

### Ordering flow (design)

A personal agent can never spend your money on its own: placing an order always pauses for your approval in the app.
This is the design the store agents and orchestrator are being ported to.

```mermaid
sequenceDiagram
    actor You
    participant App as Mobile app
    participant O as Orchestrator
    participant S as Store agent
    You->>App: "Two lattes from the cafe near me"
    App->>O: message + location
    O->>S: build_cart (A2A)
    S-->>O: priced cart from the store's own menu
    O-->>App: cart + total
    You->>App: "Yes, place it"
    App->>O: message
    O->>O: policy gate: needs human approval
    O-->>App: Approve this order?
    You->>App: Approve
    App->>O: approve
    O->>S: place_order (confirmed by user)
    S->>S: check reach (distance), store open, draft is fresh
    S-->>O: order confirmed
    O-->>App: Order placed
```

### Inside WalleBackendTs

WMS Core layering: each layer calls only the one below it, business rules live only in the Api layer, and an
architecture test fails the build if anyone skips a layer.

```mermaid
flowchart LR
    C["Controller<br/>one DtoApi"] --> D["DtoApi<br/>trim → normalize → validate"]
    D --> F["FlowApi<br/>multi-Api workflows"]
    D --> A["Api<br/>business rules · transactions"]
    F --> A
    A --> DAO["Dao<br/>native SQL"]
    A --> L["Client<br/>Anthropic"]
    DAO --> DB[("Postgres + PostGIS")]
```

## Repositories

| Repo | What it is | Stack |
|------|------------|-------|
| [**WalleBackendTs**](https://github.com/WalleLabs/WalleBackendTs) | The server. Registry (discovery, geofencing, "which store am I in?", agent cards, menus), schema migrations and seeding; store agents and the personal agent are being ported. WMS Core layering: Controller → DtoApi → Flow → Api → Dao. | Node, TypeScript, Fastify, pg, zod |
| [**WalleDockerCompose**](https://github.com/WalleLabs/WalleDockerCompose) | One-command local stack: PostGIS + the backend + the website, plus a seed job with sample Austin stores. | Docker Compose |
| [**mobile-mono-repo**](https://github.com/WalleLabs/mobile-mono-repo) | The Expo mobile app (Explore map, store chat, orders) plus the original TypeScript prototype of the backend that WalleBackendTs replaces. | Expo, React Native, TypeScript |
| [**Docs**](https://github.com/WalleLabs/Docs) | Product, strategy, competitive research, architecture, business plan and pitch deck. Start with the numbered docs in `docs/`. | Markdown |
| [**Website**](https://github.com/WalleLabs/Website) | The public site, styled like the iOS app: landing page, a live demo that runs on the real registry, and a page for stores. | Next.js 16, TypeScript |

## Run it locally

```bash
# clone the three repos side by side
git clone https://github.com/WalleLabs/WalleBackendTs.git
git clone https://github.com/WalleLabs/Website.git
git clone https://github.com/WalleLabs/WalleDockerCompose.git
cd WalleDockerCompose
cp .env.example .env
docker compose up -d --build                # the api migrates the database on start
docker compose run --rm --build seed        # loads 10 sample stores in Austin
```

Website on `:3000`, registry on `:8787`. Open http://localhost:3000/explore for the live demo, or point the mobile
app at the registry and set the simulator location to Austin (`30.2687,-97.7428`).

## Principles

- **Money rules live in code, not in prompts.** Prices come only from a store's menu; ordering is gated by distance
  rules the owner sets; a personal agent can never place an order without the human approving it in the app.
- **Calm by default.** One front door per place, intent-first app, private personal agents.
- **Open protocols.** Store agents publish A2A agent cards with a location extension, so they're discoverable by anyone.
- **Location is used, not kept.** "Which store am I in?" is answered per request and never stored.

## Status & roadmap

| Area | State |
|------|-------|
| Discovery, geofencing, presence | Working |
| Store agents, menu, priced cart, orders | Being ported to the TypeScript backend |
| Personal agent + human approval | Being ported to the TypeScript backend |
| Mobile app (Explore, chat, orders shell) | Working prototype (chat returns once store agents are ported) |
| Website (landing, live demo, for stores) | Working, not deployed yet |
| Payments, POS sync, production auth, MCP connections | Planned |
