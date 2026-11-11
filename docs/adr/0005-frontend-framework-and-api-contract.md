---
status: accepted
date: 2026-11-11
decision-makers: Gustav Gybäck
---

# Frontend framework and API contract — React 19 SPA with Vite, ASP.NET Core BFF and OpenAPI TypeScript client

<!-- Related issue: #4 -->

## Context and Problem Statement

The [TeamGym](https://www.europeangymnastics.com/discipline/teamgym/intro) competition platform needs a web frontend for competition organisers, club administrators, coaches, and (in later phases) judges and competition-day officials, together with a strict contract between the frontend and the `.NET 10` modular monolith backend ([ADR-0004](0004-backend-runtime-and-architectural-style.md)).

We need to decide:

1. **The frontend framework and runtime model** — whether to build a React Single-Page Application (SPA) with Vite served/fronted by the ASP.NET Core host, a server-rendered Next.js application running on a separate Node.js runtime, or a Blazor application.
2. **The authentication boundary between browser and backend** — how the browser authenticates without exposing OAuth/OIDC tokens to JavaScript.
3. **The API contract and client generation pipeline** — how the frontend and `.NET 10` backend stay type-safe and incapable of silently drifting apart.
4. **The supporting frontend stack** — UI component primitives, routing, server/form state, internationalisation (`sv` and `en`), and how we handle SEO/SSR and competition-floor network resilience.

## Decision Drivers

- **One source of truth for TeamGym business logic:** All domain invariants (team composition across **Women**, **Men**, **Mixed** and **Open** sections; gymnast counts and rotation scheduling across **Floor**, **Tumble** and **Trampet**; application deadlines) must live in the `.NET 10` backend ([ADR-0004](0004-backend-runtime-and-architectural-style.md)), never duplicated or split into a parallel server-side JavaScript layer.
- **Type-safe, generated API contract enforced in CI:** Frontend TypeScript code and backend ASP.NET Core Minimal APIs must never silently drift apart ([ADR-0003](0003-engineering-workflow-and-quality-gates.md)).
- **Simple, low-cost hosting for a solo maintainer:** Ideally operate a single production application runtime and a single deployable container image ([ADR-0002](0002-repository-strategy.md)) rather than managing and paying for both a `.NET` container and a persistent Node.js SSR container.
- **Secure browser authentication (BFF pattern):** Avoid storing access or refresh tokens in `localStorage` or exposing them to browser JavaScript; rely on `HttpOnly`, `Secure`, `SameSite` cookies managed by the Backend-for-Frontend (BFF).
- **Accessible (WCAG 2.2 AA) and mobile-responsive:** TeamGym coaches, team leaders, and officials frequently use phones and tablets on the competition floor next to the **Floor**, **Tumble**, and **Trampet** apparatus.
- **Internationalisation from day one:** Support at least Swedish (`sv`, *truppgymnastik*) and English (`en`, aligned with official [European Gymnastics TeamGym](https://www.europeangymnastics.com/discipline/teamgym/intro) terminology) without retrofitting string extraction later.

## Considered Options

### 1. Frontend Framework, Hosting and BFF Model

- **Option F1:** **React 19 SPA with Vite + TypeScript**, hosted with **`TeamGym.Arena.Api` acting as the Backend-for-Frontend (BFF)** and serving the compiled static SPA bundle in production (single production runtime and single container image).
- **Option F2:** **Next.js (App Router)** with React Server Components and a Node.js production server running alongside the `.NET 10` API.
- **Option F3:** **Blazor (Blazor Web App / WebAssembly)** — keeping both frontend and backend in C#/.NET.

### 2. API Contract and Client Generation

- **Option C1:** **REST + OpenAPI 3.1** generated at build time by ASP.NET Core (`Microsoft.AspNetCore.OpenApi`) with RFC 9457 `ProblemDetails`, consumed by **`openapi-typescript` + `openapi-fetch`** and verified against drift in CI.
- **Option C2:** **REST + OpenAPI** with heavyweight code generators (**Microsoft Kiota** or **NSwag**).
- **Option C3:** **GraphQL** (Hot Chocolate on `.NET` + GraphQL Code Generator on the frontend).

### 3. Supporting Frontend Stack (UI System, Routing, State, Forms & i18n)

- **Option S1:** **Tailwind CSS + `shadcn/ui` (Radix UI primitives)** for accessible UI, **TanStack Router** + **TanStack Query** for routing and server state, **React Hook Form + Zod** for forms, and **`i18next` (`react-i18next`)** for internationalisation.
- **Option S2:** **MUI (Material UI)** or **Fluent UI React** with React Router, Redux Toolkit, and Formik.

### 4. SSR/SEO and Competition-Floor Resilience Strategy

- **Option O1:** **No SSR required for MVP; resilient client caching + automatic SignalR reconnection** for the future live Competition Runner (without complex offline write queues initially).
- **Option O2:** **Full SSR for public competition pages + offline-first PWA mutation sync (IndexedDB/Service Worker)** from the start.

## Decision Outcome

Chosen options:

1. **Framework & BFF — Option F1:** Build the frontend in `src/Web` as a **React 19 SPA with Vite and strict TypeScript**, with **`TeamGym.Arena.Api` acting as the same-origin Backend-for-Frontend (BFF)** (handling OIDC login, `HttpOnly` session cookies, and serving the built static assets from a single production container).
2. **API contract — Option C1:** Use **REST over HTTP with OpenAPI 3.1** emitted at build time by `.NET 10` (`Microsoft.AspNetCore.OpenApi`) and RFC 9457 `ProblemDetails`, paired with **`openapi-typescript` and `openapi-fetch`** and a mandatory CI contract-drift gate.
3. **Supporting stack — Option S1:** Standardise on **Tailwind CSS + `shadcn/ui` (Radix UI)**, **TanStack Router**, **TanStack Query**, **React Hook Form + Zod**, and **`i18next` (`react-i18next`)** with `sv` and `en` locales from day one.
4. **SSR & offline stance — Option O1:** **No Node.js SSR runtime** (public competition pages use clean URLs and lightweight OpenGraph metadata served by the ASP.NET Core host where needed); the future live **Competition Runner** uses **TanStack Query persistence + SignalR automatic reconnect** rather than offline multi-writer conflict resolution.

### 1. Frontend Architecture and the ASP.NET Core BFF Model

- **Single production runtime and container:**
  - During the container build ([ADR-0002](0002-repository-strategy.md)), a multi-stage `Containerfile` builds the Vite frontend (`npm ci && npm run build` in `src/Web`) and copies the hashed static bundle (`dist/`) into `TeamGym.Arena.Api`'s `wwwroot/`.
  - In production, `TeamGym.Arena.Api` serves the static SPA assets (with immutable long-lived cache headers on hashed bundles) alongside `/api/v1/*` and `/bff/*` on the **exact same origin**. There is no separate Node.js server to run, patch, scale, or pay for in Azure.
- **Local development with `.NET Aspire`:**
  - Locally, `dotnet run --project src/AppHost` ([ADR-0004](0004-backend-runtime-and-architectural-style.md)) starts the Vite dev server with instant Hot Module Replacement (HMR) and proxies `/api/*`, `/bff/*`, and `/hubs/*` requests to `TeamGym.Arena.Api`.
- **Secure authentication via the BFF pattern:**
  - The browser **never** sees or stores OAuth2/OIDC access tokens or refresh tokens.
  - Authentication flows (`/bff/login`, `/bff/logout`, `/bff/user`) are handled by `TeamGym.Arena.Api` (detailed further in the identity ADR, #6), which sets an encrypted, `HttpOnly`, `Secure`, `SameSite=Lax` session cookie.
  - Because the SPA and `/api/v1/*` share the same origin, all API calls automatically include the session cookie with zero CORS configuration and built-in CSRF protection (anti-forgery header / `SameSite` policy).

### 2. API Contract and TypeScript Client Pipeline

- **Code-first in `.NET 10`, contract-verified in CI:**
  - Every Minimal API endpoint in `TeamGym.Arena.Modules.*` declares explicit request/response DTOs and `TypedResults` (`Ok<TResponse>`, `Created<TResponse>`, `ProblemHttpResult`, `ValidationProblem`).
  - `Microsoft.AspNetCore.OpenApi` and `Microsoft.Extensions.ApiDescription.Server` export a deterministic **OpenAPI 3.1** specification (`openapi.json`) at build time without needing a running database container.
- **Zero-bloat generated TypeScript client (`openapi-typescript` + `openapi-fetch`):**
  - `openapi-typescript` converts `openapi.json` into pure TypeScript type definitions (`src/Web/src/shared/api/schema.d.ts`) with zero runtime JavaScript footprint.
  - `openapi-fetch` wraps the native browser `fetch` API with compile-time path, query parameter, request body, and status-code inference directly from `schema.d.ts`.
  - If a backend developer or AI coding agent renames a property on a C# response record or changes a route parameter, `tsc --noEmit` fails immediately in `src/Web`.
- **Standardised error handling (RFC 9457 `ProblemDetails`):**
  - All API errors and validation failures return standard `application/problem+json` responses (`ProblemDetails` and `HttpValidationProblemDetails`).
  - A shared client interceptor maps field-level validation errors directly onto React Hook Form fields and translates domain error codes via `i18next`.
- **CI drift gate:**
  - In `.github/workflows/ci.yml` ([ADR-0003](0003-engineering-workflow-and-quality-gates.md)), CI builds the backend, regenerates `openapi.json` and `schema.d.ts`, and runs `git diff --exit-code`. Any pull request that modifies an API contract without regenerating and committing the updated TypeScript types fails CI automatically.

### 3. Supporting Frontend Stack and Structure (`src/Web`)

The frontend under `src/Web/` mirrors the modular bounded contexts of the backend ([ADR-0004](0004-backend-runtime-and-architectural-style.md)):

```text
src/Web/
├── public/
├── src/
│   ├── app/                          # App entrypoint, providers, global styles & TanStack Router setup
│   ├── features/                     # Feature modules aligned with backend bounded contexts
│   │   ├── auth/                     # BFF session state, sign-in/sign-out flows & route guards
│   │   ├── organisations/            # Clubs, coaches, gymnasts & TeamGym teams (Women, Men, Mixed, Open)
│   │   ├── competitions/             # Competition configuration, sections, classes & disciplines
│   │   ├── applications/             # Team competition applications & organiser review workflows
│   │   └── scheduling/               # Rotation schedule builder & viewer (Floor, Tumble, Trampet)
│   ├── shared/
│   │   ├── api/                      # Generated OpenAPI types (schema.d.ts) & configured openapi-fetch client
│   │   ├── components/ui/            # Accessible shadcn/ui + Radix UI primitives
│   │   ├── i18n/                     # i18next configuration and locale files (sv/, en/)
│   │   └── lib/                      # Shared utilities, date/time formatting, accessibility helpers
│   └── main.tsx
├── index.html
├── package.json
├── tsconfig.json
└── vite.config.ts
```

- **UI components & accessibility (`Tailwind CSS` + `shadcn/ui` / `Radix UI`):**
  - `shadcn/ui` (built on unstyled, WAI-ARIA-compliant **Radix UI** primitives and **Tailwind CSS**) lives directly in `src/Web/src/shared/components/ui/` as editable source code rather than an opaque black-box dependency.
  - Gives full keyboard navigation, screen-reader announcements, focus trapping, and WCAG 2.2 AA colour contrast out of the box, while making dense TeamGym views (such as multi-column rotation grids across **Floor**, **Tumble** and **Trampet**) easy to tailor for both desktop monitors and mobile phones on the competition floor.
- **Routing (`TanStack Router`):**
  - Provides 100% type-safe route parameters, search/filter query parameters (e.g. filtering a competition schedule by **Section**: `Women`, `Men`, `Mixed`, `Open`, or **Discipline**: `Floor`, `Tumble`, `Trampet`), and route loaders integrated with TanStack Query.
- **Server state (`TanStack Query`):**
  - Manages caching, background refetching, optimistic UI updates, and cache invalidation on mutations. Eliminates the need for a global Redux store for server data.
- **Forms & client validation (`React Hook Form` + `Zod`):**
  - Handles multi-step and dynamic forms (such as registering a TeamGym team of 8–10 gymnasts, validating gender ratios for **Mixed** [50%/50%] vs **Women** / **Men** / **Open**, or configuring competition classes) with accessible inline error messages (`aria-invalid`, `aria-describedby`), while the `.NET` backend remains the authoritative validator.
- **Internationalisation (`i18next` + `react-i18next`):**
  - Configured from the initial commit with two locales:
    - **`en` (English):** Canonical European Gymnastics TeamGym terminology (**Floor**, **Tumble**, **Trampet**; **Women**, **Men**, **Mixed**, **Open**).
    - **`sv` (Swedish):** Swedish TeamGym (*truppgymnastik*) terminology (*Fristående*, *Tumbling*, *Trampett*; *Dam*, *Herr*, *Mixed*, *Öppen*).
  - No user-facing string may be hardcoded in JSX components; ESLint rules and PR reviews enforce translation keys, and formatting of dates, times, and numbers uses standard `Intl` APIs.

### 4. Answers to Open Questions (SEO/SSR and Offline Support)

- **Do any pages need SEO or server rendering?**
  - **No Node.js SSR runtime is needed.** The core MVP workflows (club administration, team registration, competition applications, schedule authoring) are authenticated application workflows where SSR adds operational complexity without benefit.
  - Public read-only pages (such as a published competition overview or rotation schedule shared via link) use clean, bookmarkable URLs and fast client rendering against cached public GET endpoints. When rich link previews (OpenGraph `<meta>` tags for chat/social sharing) are desired for a competition URL, `TeamGym.Arena.Api` can inject dynamic `<meta>` tags into `index.html` on initial document request — achieving rich link previews without running Next.js or Node.js in production.
- **Will the future Competition Runner need offline support?**
  - **Resilient network handling, not full offline-first multi-writer sync:** On competition day, sports halls can have spotty Wi-Fi/mobile coverage. However, a live TeamGym competition involves multiple concurrent actors (competition director, floor/apparatus marshals, judges across **Floor**, **Tumble** and **Trampet**). True offline multi-writer state synchronisation introduces severe conflict-resolution hazards.
  - Instead, the SPA will use **TanStack Query cache persistence** so coaches and officials can continue viewing the current rotation schedule and start lists on their phones during brief Wi-Fi drops, paired with **ASP.NET Core SignalR automatic reconnection and state reconciliation** when the live **Competition Runner** (`Runner`) and **Score Management** (`Scoring`) modules are introduced ([ADR-0004](0004-backend-runtime-and-architectural-style.md)).

### Consequences

- Good, because operating a single `.NET 10` container that serves both the API/BFF and the compiled React SPA keeps Azure hosting simple, cheap, and scale-to-zero friendly ([ADR-0002](0002-repository-strategy.md), #8).
- Good, because same-origin `HttpOnly` cookies via the ASP.NET Core BFF eliminate browser token theft risks (XSS token exfiltration) and CORS complexity (#6).
- Good, because `openapi-typescript` + `openapi-fetch` + the CI drift gate make it impossible for a PR to break the frontend/backend contract silently.
- Good, because React 19, Vite, Tailwind CSS, `shadcn/ui`, TanStack Router/Query, React Hook Form + Zod, and `i18next` have massive ecosystem adoption and are idiomatic for both human contributors and AI coding agents.
- Good, because building `sv` and `en` internationalisation and WCAG 2.2 AA accessibility (`vitest-axe` + Playwright `axe-core`) into the scaffold avoids an expensive retrofit later.
- Bad, because `shadcn/ui` components live in the repository under `src/Web/src/shared/components/ui/` and are maintained as part of our codebase (mitigated by keeping primitives thin and encapsulated).

### Confirmation

- `.github/workflows/ci.yml` ([ADR-0003](0003-engineering-workflow-and-quality-gates.md)) enforces on every pull request:
  - OpenAPI contract regeneration and `git diff --exit-code` verification so `schema.d.ts` never drifts from `TeamGym.Arena.Api`.
  - `tsc --noEmit` (strict TypeScript type-checking), `eslint --max-warnings=0`, and `prettier --check`.
  - Frontend unit and component tests via **Vitest + React Testing Library**, including automated accessibility assertions with **`vitest-axe`**.
  - End-to-end browser tests via **Playwright** (`tests/E2E`) including **`@axe-core/playwright`** WCAG 2.2 AA scans on key user journeys.

## Pros and Cons of the Options

### Frontend Framework, Hosting and BFF Model

#### Option F1: React 19 SPA with Vite + ASP.NET Core BFF (Single Production Runtime)

- Good, because all server-side business logic, authentication (`HttpOnly` cookies), and API endpoints live in one `.NET 10` process (`TeamGym.Arena.Api`).
- Good, because production deployment requires only a single container image with no Node.js runtime in production.
- Good, because Vite provides near-instant local dev startup and HMR, and integrates cleanly with `.NET Aspire` (`src/AppHost`).
- Neutral, because initial page load downloads the SPA bundle (kept small via route-based code splitting in TanStack Router and Vite).

#### Option F2: Next.js (App Router) with Server Rendering

- Good, because React Server Components and built-in SSR provide first-paint HTML rendering for public pages.
- Bad, because it requires operating, securing, and paying for a second production server runtime (Node.js) alongside `.NET 10`.
- Bad, because it splits backend concerns across two languages and runtimes (a Next.js BFF/server-action layer in TypeScript vs `.NET` Minimal APIs), creating ambiguity for AI coding agents over where validation, auth checks, and data fetching belong.
- Bad, because frequent Next.js App Router caching and bundler changes create unnecessary maintenance churn for an application that is primarily an interactive administration tool.

#### Option F3: Blazor (Blazor Web App / WebAssembly)

- Good, because C# types and domain validation logic can be shared directly between client and server without TypeScript code generation.
- Bad, because Blazor WebAssembly has a larger initial payload and slower cold start on mobile connections inside sports halls, while Blazor Server requires a persistent SignalR circuit for every UI interaction.
- Bad, because the ecosystem for accessible, mobile-first headless UI primitives and AI agent fluency is significantly stronger in React + TypeScript + Tailwind than in Blazor.

### API Contract and Client Generation

#### Option C1: REST + OpenAPI 3.1 + `openapi-typescript` & `openapi-fetch`

- Good, because `.NET 10` (`Microsoft.AspNetCore.OpenApi`) natively generates OpenAPI 3.1 schemas from Minimal API `TypedResults` with minimal attributes or ceremony.
- Good, because `openapi-typescript` generates pure TypeScript types (0 kB runtime overhead) and `openapi-fetch` provides full path/method/status type inference over standard `fetch`.
- Good, because RFC 9457 `ProblemDetails` is an IETF standard supported natively by ASP.NET Core.
- Neutral, because TanStack Query hooks are written as thin, explicit wrappers around the typed `openapi-fetch` client (or optionally via `@hey-api/openapi-ts` / `openapi-react-query` where helpful).

#### Option C2: REST + OpenAPI with Kiota or NSwag

- Good, because both tools come from or have long histories in the `.NET` ecosystem.
- Bad, because Kiota generates verbose class hierarchies and request-builder abstractions that feel un-idiomatic in modern React + TanStack Query codebases.
- Bad, because NSwag has slower adoption of OpenAPI 3.1 features and generates large, monolithic runtime client files.

#### Option C3: GraphQL (Hot Chocolate + GraphQL Code Generator)

- Good, because clients can request arbitrary field subsets in a single query.
- Bad, because adding a GraphQL schema layer on top of `.NET` vertical slices duplicates contract definitions and complicates HTTP caching, file uploads (future music submission), and simple REST semantics.
- Bad, because vertical slice endpoints in a modular monolith already tailor their response DTOs to the exact needs of the UI.

### Supporting Frontend Stack

#### Option S1: Tailwind CSS + `shadcn/ui` (Radix UI) + TanStack Router/Query + React Hook Form & Zod + `i18next`

- Good, because Radix UI primitives provide battle-tested WAI-ARIA accessibility and keyboard support out of the box, while Tailwind CSS makes responsive competition-floor mobile layouts straightforward.
- Good, because TanStack Router + TanStack Query provide end-to-end type safety for URL parameters, search filters, and server cache state.
- Good, because `i18next` is the industry standard for React internationalisation and cleanly separates `sv` and `en` translation files.

#### Option S2: MUI or Fluent UI + React Router + Redux Toolkit

- Good, because MUI and Fluent UI provide pre-styled enterprise widget suites.
- Bad, because overriding CSS-in-JS styles for custom TeamGym schedule matrices and mobile competition-floor views adds bundle weight and runtime overhead.
- Bad, because Redux Toolkit introduces unnecessary boilerplate when server state is already managed by TanStack Query.

## More Information

- Related issue: [#4](https://github.com/gyback/teamgym-arena/issues/4)
- Parent epic: [#9](https://github.com/gyback/teamgym-arena/issues/9)
- Related ADRs and decisions:
  - [ADR-0001 Record architecture decisions using MADR](0001-record-architecture-decisions.md)
  - [ADR-0002 Repository strategy — public app monorepo, private IaC repo and artefact hand-off](0002-repository-strategy.md)
  - [ADR-0003 Engineering workflow and quality gates for agent-assisted development](0003-engineering-workflow-and-quality-gates.md)
  - [ADR-0004 Backend runtime and architectural style — .NET 10 modular monolith with vertical slices](0004-backend-runtime-and-architectural-style.md)
  - [#6 Identity, authentication and authorisation](https://github.com/gyback/teamgym-arena/issues/6) (concrete OIDC provider and ASP.NET Core BFF `/bff/*` authentication endpoints)
  - [#8 Cloud hosting platform and Infrastructure as Code](https://github.com/gyback/teamgym-arena/issues/8) (hosting the single combined API + SPA container image on Azure)
- Follow-up implementation issues:
  - [#21 Scaffold React 19 + Vite frontend (`src/Web`) with Tailwind, shadcn/ui, TanStack Router/Query and i18next](https://github.com/gyback/teamgym-arena/issues/21)
  - [#22 Add OpenAPI 3.1 build-time export and TypeScript client generation (`openapi-typescript` + `openapi-fetch`) in CI](https://github.com/gyback/teamgym-arena/issues/22)
