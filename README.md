# Laravel Coding Guides

> A practical, opinionated standard for how we build Laravel applications — the
> conventions, architecture, and patterns we apply on every project so the code
> stays readable, consistent, testable, and safe to change.

Target stack: **PHP 8.4+, Laravel 13+, Pest, Pint, Livewire, Filament.**

These guides distil a real, production multi-tenant SaaS build into a reusable
reference. They favour a clean **layered architecture**, strict typing, and a
hard rule that all data access goes through **repositories**. Read
[Philosophy](docs/philosophy.md) first, then dip into the layer you're working in.

---

## The shape of an application

```
Route
  └─ Controller            HTTP glue only — no logic
       └─ Form Request     validation + authorization
       └─ Action           one write / orchestration
            └─ Service      reusable capability / external integration
                 └─ Repository   the ONLY place a model is touched
                      └─ Eloquent / Database
```

Parallel entry points — **Livewire** components, queued **Jobs**, console
**Commands**, the **Filament** admin — all flow down the same layers.
Dependencies point downward and are wired through **interfaces**.

## Core principles

- **SOLID** — single responsibility per class; depend on abstractions; small interfaces.
- **DRY** — never write the same logic twice; extract it into one Action/Service.
- **Types everywhere** — `declare(strict_types=1)`, typed params + returns, `final` classes.
- **Thin HTTP layer** — controllers delegate; validation lives in Form Requests.
- **Repositories own data access** — application code never touches a model directly.
- **Fail loudly** — throw typed exceptions; no silent fallbacks unless they're a tested requirement.
- **Tests are part of done** — a unit test per class, a feature test per feature; nothing hits the network.
- **Enforced, not aspirational** — Pint + CI keep the mechanical rules honest.

---

## Contents

### Foundations
- [Philosophy & how to use this guide](docs/philosophy.md)
- [Application layers & where logic goes](docs/architecture.md)
- [Code style & language rules](docs/code-style.md)
- [SOLID principles in Laravel](docs/solid.md)
- [DRY & reuse](docs/dry.md)

### The layers
- [Skinny controllers](docs/controllers.md)
- [Validation & Form Requests](docs/form-requests.md)
- [Actions (single-purpose write operations)](docs/actions.md)
- [Services (reusable domain logic & integrations)](docs/services.md)
- [Repositories — the only place models are touched](docs/repositories.md)
- [Models & Eloquent](docs/models-eloquent.md)
- [DTOs & value objects](docs/dtos-value-objects.md)
- [Enums](docs/enums.md)

### Surfaces
- [Livewire components](docs/livewire.md)
- [Filament (admin panels)](docs/filament.md)
- [Jobs, commands & scheduling](docs/jobs-commands-scheduling.md)

### Cross-cutting
- [Naming conventions](docs/naming-conventions.md)
- [Testing (Pest: unit + feature)](docs/testing.md)
- [Configuration & secrets](docs/config-and-secrets.md)
- [Error handling & exceptions](docs/error-handling.md)

### Tooling
- [Tooling & CI](docs/tooling-and-ci.md)
- [Quick reference & PR checklist](docs/checklists.md)

---

## How to use this

- **Starting a feature?** Skim [Architecture](docs/architecture.md) to place each
  piece of logic in the right layer.
- **In review?** Run the [PR checklist](docs/checklists.md).
- **New to the stack?** Read Foundations top to bottom, then the layer pages.
- **Enforcement:** copy `pint.json` and the CI workflow from
  [Tooling & CI](docs/tooling-and-ci.md); run `composer check` before every commit.
