# IH – SS26 SWEN2 – Team 03 — Review Feedback

**Repository:** [drumnadrochit/IH-ss26swen2team3](https://github.com/drumnadrochit/IH-ss26swen2team3)
**Upstream:** [RickyRAV/ss26swen2team3](https://github.com/RickyRAV/ss26swen2team3)
**Live:** https://tourplanner.w11core.cc
**Reviewed branch:** `main` (commit `37e190a`)
**Stack (actual):** React 19 (Vite + TanStack Router/Query/Form) · C# ASP.NET 8 · PostgreSQL · Docker
**Language composition:** C# 53% · TypeScript 43.6% · Dockerfile 1.6% · CSS 1.3% · Other 0.5%

---

## TL;DR

The backend is in very good shape — a clean, strict 3-tier architecture (Api → BL → DAL → Models) with interfaces, DI, EF Core repositories and well-separated exception types. The MVVM folder layout on the frontend is also clean (`models/ · viewmodels/ · views/ · components/ · stores/`) and the must-have features for Tours and Tour Logs are all implemented (CRUD, list/detail views, real map via Leaflet, validation via Zod + FluentValidation-style validators on the backend).

**However, there is one blocking deviation from the must-haves:** the frontend is built with **React**, not **Angular**. According to the brief ("Uses Angular as frontend framework"), this is a hard requirement and currently not fulfilled.

---

## 1. Architecture — 3-Tier & SOLID

### 1.1 3-Tier separation (Backend, C#) — ✅ Excellent

The solution `apps/api/TourPlanner.sln` is split into the four classic tiers, exactly as expected:

```
apps/api/
├── TourPlanner.Api      → Controllers, DTOs, Validators, Middleware (Presentation)
├── TourPlanner.BL       → Services + Interfaces, BL Exceptions     (Business Logic)
├── TourPlanner.DAL      → Repositories, EF Core, DbContext, Migrations, DAL Exceptions (Data Access)
├── TourPlanner.Models   → Domain entities / enums                   (Domain)
└── TourPlanner.Tests    → NUnit tests
```

The README explicitly states and enforces the layering rules:

> Controllers never touch repositories. BL never touches EF Core directly.

This is exactly the kind of strict 3-tier separation the course expects. ✅

### 1.2 SOLID — ✅ Good, with room to grow

Observed positives:

- **SRP (Single Responsibility):** Each tier has one job; services (`TourService`, `TourLogService`, `AuthService`, `ImageService`, `OrsService`) are split by domain concern.
- **OCP / DIP (Open/Closed, Dependency Inversion):** Every service ships with its interface — `ITourService`, `ITourLogService`, `IAuthService`, `IImageService`, `IOrsService` — and is registered via DI in `Program.cs`. Controllers depend on the abstraction, not the implementation. This is exactly the pattern from refactoring.guru's *Dependency Injection* article.
- **ISP (Interface Segregation):** Service interfaces are small and domain-focused (e.g. `IImageService` is intentionally tiny — only image concerns).
- **LSP:** No surprising base-class hierarchies; repositories and services are composed, not deeply inherited — good.

Suggestions to push SOLID further:

- Consider extracting a thin **mapping layer** (e.g. `IMapper` / Mapperly / explicit mapping classes) so that `Service` classes don't manually translate between domain models and DTOs.
- The `DataSeeder.cs` in `TourPlanner.Api` (~7.8 KB) is doing a lot. Move seeding into BL or its own seeder class to keep `Api` purely presentation-layer.
- A few "patterns from refactoring.guru" that would fit naturally here:
  - **Strategy** for transport-type-specific behaviour (Car/Bicycle/Hiking/Running/Vacation routing differences via `OrsService`).
  - **Factory** for creating `Tour` / `TourLog` aggregates with their computed fields (distance, estimated time).
  - **Repository** pattern is already present in `TourPlanner.DAL/Repositories` — good.

### 1.3 Frontend layering — ⚠️ Right idea, wrong framework

The folder structure under `apps/web/src/` is textbook MVVM:

```
apps/web/src/
├── api/          # API client functions
├── models/       # TypeScript interfaces (Model layer)
├── viewmodels/   # TanStack Query / Form hooks (ViewModel layer)
├── views/        # Components (View layer)
├── stores/       # Zustand — auth state only
└── components/   # Shared / reusable UI
```

The README even codifies the rule:

> MVVM rule: **Views only import from `viewmodels/`**, never from `api/` directly.

That's a great convention. The problem is that this is implemented on top of **React 19**, not Angular (see §2.1).

---

## 2. Must-Have Features

### 2.1 Uses Angular as frontend framework — ❌ Not fulfilled (blocking)

`apps/web/package.json` shows:

- `react` `^19.2.4`, `react-dom` `^19.2.4`
- `@tanstack/react-router`, `@tanstack/react-query`, `@tanstack/react-form`, `@tanstack/react-table`
- `vite` `^8.0.4`
- No `@angular/*` packages anywhere.

The course brief explicitly requires **Angular** as the frontend framework. The current implementation is a (very clean) React + TanStack stack instead.

**Action required:** clarify with the lecturer whether this deviation is accepted. If not, the entire `apps/web/` module must be ported to Angular (Angular CLI project, Angular components, Angular services as ViewModels, `RxJS`/`signals` for reactivity, Angular `FormGroup`/`FormControl` for forms, Angular Router instead of TanStack Router, etc.). The good news: because the MVVM separation is already strict, the **models/** and the API contracts can be reused 1:1, and most viewmodels translate to Angular services with relatively low friction.

### 2.2 Uses MVVM for UI — ✅ Implemented (in React)

- **Model:** `apps/web/src/models/tour.ts`, `tourLog.ts`, `user.ts` — pure TS interfaces.
- **ViewModel:** `apps/web/src/viewmodels/` — `useTourListViewModel`, `useTourDetailViewModel`, `useTourFormViewModel`, `useTourLogListViewModel`, `useTourLogFormViewModel`, `useAuthViewModel`.
- **View:** `apps/web/src/views/tours/{TourListView,TourDetailView,TourFormDialog}.tsx` and `apps/web/src/views/tourLogs/{TourLogListView,TourLogFormDialog}.tsx`.

The split between data-fetching/mutation logic (viewmodels) and presentation (views) is consistently applied. ⭐
*(In Angular this would map to: components = views, services with signals/observables = viewmodels, interfaces/classes = models.)*

### 2.3 GUI in general

#### Correct data binding between UI elements and view model properties — ✅
Forms and list state are wired through dedicated viewmodel hooks (`useTourFormViewModel`, `useTourLogFormViewModel`) using `@tanstack/react-form`. Views import only from `viewmodels/`. In Angular this should be reproduced via `FormGroup` + two-way binding (`[(ngModel)]`) or reactive forms.

#### UI responds to window size changes — ✅
Tailwind CSS v4 is used (`@tailwindcss/vite`), and Radix UI primitives are responsive. Verify on the live site that the Tour list ↔ Tour detail ↔ Tour log columns behave correctly on narrow viewports.

#### Defines reusable UI Component — ✅
Reusable components are present:
- `apps/web/src/components/ConfirmDialog.tsx` — generic confirm dialog.
- `apps/web/src/components/DataTable.tsx` — generic table built on `@tanstack/react-table`.
- `apps/web/src/components/TourMap.tsx` — Leaflet map (not just a placeholder — actual interactive map).
- `apps/web/src/components/ui/*` — shadcn/Radix-based primitives.

### 2.4 Tours

| Sub-requirement | Status | Evidence |
|---|---|---|
| Create / modify / delete tour | ✅ | `TourFormDialog.tsx`, `useTourFormViewModel.ts`, `TourService.cs`, `TourController` (Api) |
| Required attributes incl. image | ✅ | `Tour` model has `name`, `description`, `from`, `to`, `transportType`, `imagePath`, `distance`, `estimatedTimeSeconds`, `popularity`, `childFriendliness`. `ImageService.cs` handles image upload/storage. |
| Managed in a list view | ✅ | `TourListView.tsx` (uses `DataTable`) |
| Tour Details show all attributes + map placeholder | ✅ (exceeds requirement) | `TourDetailView.tsx` + `TourMap.tsx` (real Leaflet map, not just a placeholder) |
| Validates user input (no crash) | ✅ | Zod on the client (`zod ^4.3.6`), `TourValidators.cs` on the server, plus a `Middleware/` folder for global exception handling. |

Minor suggestions:
- `CreateTourPayload` only requires `name/description/from/to/transportType`, while `UpdateTourPayload` allows `distance/estimatedTimeSeconds/routeInformation`. Document explicitly that these extra fields are *server-computed on create* via `OrsService`, so it's clear why they're not in the create payload.
- Add a confirm step (already have `ConfirmDialog`) for destructive actions on Tours if not yet wired in every entry point.

### 2.5 Tour Logs

| Sub-requirement | Status | Evidence |
|---|---|---|
| Create / modify / delete tour log | ✅ | `TourLogFormDialog.tsx`, `useTourLogFormViewModel.ts`, `TourLogService.cs` |
| Required attributes | ✅ | `TourLog` model has `dateTime`, `comment`, `difficulty (Easy/Medium/Hard)`, `totalDistanceKm`, `totalTimeSeconds`, `rating (One..Five)`, plus `tourId`, `createdAt`, `updatedAt`. |
| Listed for the selected tour with all attributes | ✅ | `TourLogListView.tsx` shown inside `TourDetailView.tsx` |
| Validates user input (no crash) | ✅ | `TourLogValidators.cs` on the backend, Zod on the frontend. |

Minor suggestions:
- Consider showing a small chart of log ratings/times over time using the already-installed `recharts` — would be a nice "nice-to-have" for the next milestone.
- The `Rating` enum (`One`..`Five`) is mapped via `RATING_TO_NUMBER` / `NUMBER_TO_RATING`. That's fine, but consider exposing a `ratingValue: number` directly on the API to remove the dual representation.

### 2.6 Protocol — ✅ Present, please verify wireframes are referenced

- `Docs/TourPlanner_Protocol.pdf` (~1.28 MB) is committed.
- `Docs/wireframes/` contains: `Login.png`, `Register.png`, `Tours Empty State.png`, `Tours Detail State.png`, `Tours Detail State-1.png`, `Tours Detail State-2.png`, `Tours Detail State-3.png`, plus subfolders `Wireframe 3 — Main View/` and `Wireframe 4 — Main View/`.

Checklist for the protocol:
- ☐ Wireframes are **embedded/referenced** inside the PDF, not just sitting next to it.
- ☐ A short **UX narrative** explains the user journey (login → tour list → tour detail → log creation).
- ☐ The protocol notes the framework decision and (importantly) the deviation from the Angular requirement.

---

## 3. Cross-Cutting Observations

### Strengths
- **Dev experience is excellent.** One-command Docker setup (`docker compose up`), auto-applied EF migrations, demo seed (`demo@tourplanner.dev` / `Demo123!`), PR previews on `pr-{N}.tourplanner.w11core.cc`, Swagger at `/api/v1/swagger`, Postman collection committed.
- **Strict layering enforced in code review** (`README.md` documents the rule, repo layout backs it up).
- **Tests scaffolded:** `TourPlanner.Tests/` NUnit project + `pnpm typecheck` / `pnpm build` for the frontend.
- **Branching/PR convention is explicit and clean** in the README.
- **Real map (Leaflet)** instead of the required placeholder — a clear "exceeds expectations".

### Risks / Issues
1. **Frontend framework mismatch (Angular vs. React)** — see §2.1. This is the most important item to resolve.
2. **TypeScript version pinned to `~6.0.2`** in `apps/web/package.json` — that version does not exist on npm (TS is currently 5.x). This is almost certainly a typo and will break a fresh `pnpm install`. Please change to `~5.6.0` or `^5.7.0`.
3. **React pinned to `^19.2.4`** — a very fresh major. If a port to Angular is required anyway, this becomes moot.
4. **`DataSeeder.cs` lives in the API tier.** Consider moving to BL or a dedicated `TourPlanner.Seeding` project to keep the API tier "controllers + DTOs" only.
5. **`stores/` (Zustand)** is used "for auth state only". Document this constraint in code (e.g. a `README.md` inside `stores/`) so future contributors don't widen it.

---

## 4. Recommendation per Must-Have

| Must-Have | Status |
|---|---|
| Uses Angular as frontend framework | ❌ Not fulfilled — currently React |
| Uses MVVM for UI | ✅ (in React; conceptually correct) |
| GUI: correct data binding | ✅ |
| GUI: responds to window size | ✅ |
| GUI: defines reusable UI Component | ✅ |
| Tours: create / modify / delete | ✅ |
| Tours: required attributes incl. image | ✅ |
| Tours: managed in list view | ✅ |
| Tours: detail view incl. map placeholder | ✅ (real map) |
| Tours: validates user input | ✅ |
| Tour Logs: create / modify / delete | ✅ |
| Tour Logs: required attributes | ✅ |
| Tour Logs: listed per tour | ✅ |
| Tour Logs: validates user input | ✅ |
| Protocol: describes UX incl. wireframes | ✅ (PDF + wireframes present — confirm content) |
| 3-tier architecture | ✅ Excellent |
| SOLID principles | ✅ Good (interfaces, DI, separation) |
| Patterns from refactoring.guru | ✅ Repository, DI; consider Strategy & Factory next |

---

## 5. Concrete Next Steps (in priority order)

1. **Clarify the Angular requirement with the lecturer.** Either get an explicit exception in writing, or plan a port:
   - Scaffold `apps/web-angular/` with Angular CLI.
   - Reuse `models/*.ts` 1:1.
   - Translate viewmodels (`useTour*ViewModel.ts`) into Angular services with `signal`s or `BehaviorSubject`s.
   - Translate views into Angular components; use reactive forms for validation.
   - Replace Leaflet usage via `@asymmetrik/ngx-leaflet` (or keep Leaflet directly).
2. **Fix the TypeScript version pin** (`~6.0.2` → `~5.6.0`) in `apps/web/package.json` to unbreak fresh installs.
3. **Confirm the protocol PDF actually contains the wireframes + UX narrative** (not just files in the same folder).
4. **Move `DataSeeder`** out of `TourPlanner.Api` into BL or a dedicated project.
5. Optional polish: add a chart of tour-log stats with `recharts` (or `ngx-charts` after the port).

---

*Reviewed by: @drumnadrochit — 2026-05-19*
