# CLAUDE.md - AI Assistant Quick Reference

> **Last Updated:** 2025-11-17
> **Project:** Cafeteria Management System
> **Version:** 3.2.0 - Frontend Guidelines Added

---

## 📋 Quick Navigation

### Project Specifications

- **Project Specs Index**: [`specs/README.md`](specs/README.md) - Complete project specifications
- **Architecture & DDD**: [`specs/01-architecture.md`](specs/01-architecture.md) - DDD principles, layers, error handling
- **Domain Model**: [`specs/02-domain-model.md`](specs/02-domain-model.md) - Entities, Value Objects, business logic

### Development Guides

- **Coding Standards**: [`docs/development/coding-standards.md`](docs/development/coding-standards.md)
- **Component Patterns**: [`docs/development/component-patterns.md`](docs/development/component-patterns.md)
- **Common Tasks**: [`docs/guides/common-tasks.md`](docs/guides/common-tasks.md)

---

## 🎯 Project Overview

**Cafeteria Management System** - A modern web application for managing cafeteria operations, built with **Next.js 16** and **strict Domain-Driven Design (DDD)** architecture.

### Current Status

- ✅ Base Next.js 16 with TypeScript
- ✅ Tailwind CSS v4 + shadcn/ui
- ✅ Prettier + ESLint configured
- 🚧 DDD structure implementation pending

---

## 🚀 Quick Start

```bash
# Install dependencies
npm install

# Start development server
npm run dev              # → http://localhost:3000

# Linting & Formatting
npm run lint             # Check for issues
npm run lint:fix         # Fix issues + format with Prettier

# Build
npm run build            # Production build
npm run start            # Run production build
```

---

## 🏛️ **CRITICAL: DDD Architecture**

**⚠️ This project STRICTLY follows Domain-Driven Design principles.**

### Four-Layer Architecture

```
┌─────────────────────────────────────────┐
│  Presentation (app/)                    │  ← UI, API Routes
├─────────────────────────────────────────┤
│  Application (src/application/)         │  ← Use Cases, DTOs
├─────────────────────────────────────────┤
│  Domain (src/domain/)                   │  ← Business Logic
├─────────────────────────────────────────┤
│  Infrastructure (src/infrastructure/)   │  ← DB, External APIs
└─────────────────────────────────────────┘
```

### Dependency Rule

**Dependencies flow INWARD only:**

```
Presentation → Application → Domain
Infrastructure → Application → Domain
```

### ✅ Golden Rules

1. **Domain Layer** - Pure business logic, NO framework dependencies
2. **Application Layer** - Orchestration only, imports Domain only
3. **Infrastructure Layer** - Implements Domain interfaces
4. **Presentation Layer** - Uses Application layer (NOT Domain directly)

### ❌ Prohibited

- ❌ Business logic in controllers/UI
- ❌ Domain importing other layers
- ❌ Anemic domain models (entities with only getters/setters)
- ❌ Direct database access from Application
- ❌ Violating layer dependencies

**📖 Full Architecture Guide**: [`specs/01-architecture.md`](specs/01-architecture.md)

---

## 📁 Codebase Structure (Simplified)

```
cafeteria-mg/
├── src/
│   ├── domain/                    # DOMAIN LAYER (Business Logic)
│   │   └── <context>/             # Bounded contexts (cafeteria, inventory, etc.)
│   │       ├── entities/          # Domain entities
│   │       ├── value-objects/     # Immutable value objects
│   │       ├── services/          # Domain services
│   │       ├── repositories/      # Repository interfaces (NOT implementations)
│   │       └── events/            # Domain events
│   ├── application/               # APPLICATION LAYER (Use Cases)
│   │   └── <context>/
│   │       ├── use-cases/         # Use case implementations
│   │       ├── dtos/              # Data transfer objects
│   │       └── services/          # Application services
│   └── infrastructure/            # INFRASTRUCTURE LAYER (External)
│       ├── persistence/           # DB repositories (Prisma, etc.)
│       ├── external-services/     # External APIs
│       └── config/                # Infrastructure config
├── app/                           # PRESENTATION LAYER (Next.js)
│   ├── (routes)/                  # Route groups
│   ├── api/                       # API routes
│   ├── components/
│   │   ├── ui/                    # shadcn/ui components
│   │   ├── features/              # Feature components (by context)
│   │   └── shared/                # Shared custom components
│   ├── layout.tsx
│   ├── page.tsx
│   └── globals.css
├── docs/                          # Development guides
│   ├── development/               # Next.js patterns & standards
│   └── guides/                    # Common tasks
├── specs/                         # Project specifications (DDD architecture, domain model, etc.)
└── public/                        # Static assets
```

**📖 Detailed Specifications**: [`specs/README.md`](specs/README.md)

---

## 🛠 Tech Stack (Core)

| Technology       | Version | Purpose                                 |
| ---------------- | ------- | --------------------------------------- |
| **Next.js**      | 16.0.3  | React framework with App Router         |
| **React**        | 19.2.0  | UI library                              |
| **TypeScript**   | 5.x     | Type safety (strict mode)               |
| **Tailwind CSS** | v4      | Utility-first styling                   |
| **shadcn/ui**    | Latest  | Component library (Radix UI + Tailwind) |
| **Prettier**     | Latest  | Code formatter                          |
| **ESLint**       | 9.x     | Linting (flat config)                   |

---

## 💻 Essential Conventions

### DDD Layer Rules

| Layer              | Can Import                     | Cannot Import                | Rules                                     |
| ------------------ | ------------------------------ | ---------------------------- | ----------------------------------------- |
| **Domain**         | ❌ Nothing                     | ALL other layers             | Pure business logic, framework-agnostic   |
| **Application**    | ✅ Domain                      | Infrastructure, Presentation | Orchestration only, no business logic     |
| **Infrastructure** | ✅ Domain, Application         | Presentation                 | Implements interfaces, all I/O operations |
| **Presentation**   | ✅ Application, Infrastructure | Domain directly              | UI/API handling, delegates to Application |

### File Naming

| File Type                  | Convention                    | Example                    |
| -------------------------- | ----------------------------- | -------------------------- |
| Pages                      | `page.tsx`                    | `app/dashboard/page.tsx`   |
| Layouts                    | `layout.tsx`                  | `app/layout.tsx`           |
| Components                 | PascalCase                    | `OrderCard.tsx`            |
| Entities                   | PascalCase                    | `Order.ts`, `Menu.ts`      |
| Value Objects              | PascalCase                    | `Price.ts`, `Email.ts`     |
| Use Cases                  | `<Verb><Entity>UseCase.ts`    | `CreateOrderUseCase.ts`    |
| DTOs                       | `<Purpose>Dto.ts`             | `CreateOrderDto.ts`        |
| Repository Interfaces      | `I<Entity>Repository.ts`      | `IOrderRepository.ts`      |
| Repository Implementations | `<Tech><Entity>Repository.ts` | `PrismaOrderRepository.ts` |

### TypeScript

- ✅ Strict mode enabled
- ✅ Use `import type` for type-only imports
- ✅ Explicit return types
- ❌ NO `any` type (use `unknown` if needed)

### React

- ✅ **All Frontend UI must be written in React** (React 19.2.0)
- ✅ Server Components by default
- ✅ Use `"use client"` only when necessary (hooks, events, browser APIs)
- ✅ Named props interfaces
- ❌ NO business logic in components

### Tailwind CSS

- ✅ Utility classes for styling
- ✅ Mobile-first responsive (`sm:`, `md:`, `lg:`)
- ✅ Dark mode support (`dark:`)
- ❌ Avoid inline `style={}` unless necessary

**📖 Full Standards**: [`docs/development/coding-standards.md`](docs/development/coding-standards.md)

---

## 🧩 Quick Code Examples

### Domain Entity (Business Logic)

```typescript
// src/domain/cafeteria/entities/Order.ts
export class Order {
  constructor(
    private readonly id: OrderId,
    private status: OrderStatus,
    private items: OrderItem[]
  ) {}

  addItem(item: OrderItem): void {
    if (this.status === OrderStatus.COMPLETED) {
      throw new InvalidOrderException("Cannot modify completed order");
    }
    this.items.push(item);
  }
}
```

### Application Use Case (Orchestration)

```typescript
// src/application/cafeteria/use-cases/CreateOrderUseCase.ts
export class CreateOrderUseCase {
  constructor(
    private orderRepository: IOrderRepository,
    private menuRepository: IMenuRepository
  ) {}

  async execute(dto: CreateOrderDto): Promise<OrderDto> {
    const menu = await this.menuRepository.findById(dto.menuId);
    const order = Order.create(menu, dto.quantity);
    await this.orderRepository.save(order);
    return OrderDto.fromEntity(order);
  }
}
```

### Infrastructure Repository (Implementation)

```typescript
// src/infrastructure/persistence/prisma/repositories/PrismaOrderRepository.ts
export class PrismaOrderRepository implements IOrderRepository {
  async save(order: Order): Promise<void> {
    await prisma.order.create({
      data: this.toSchema(order),
    });
  }
}
```

### Presentation API Route (Controller)

```typescript
// app/api/orders/route.ts
import { CreateOrderUseCase } from "@/src/application/cafeteria/use-cases/CreateOrderUseCase";

export async function POST(request: NextRequest) {
  const useCase = new CreateOrderUseCase(orderRepo);
  const dto = await request.json();
  const result = await useCase.execute(dto);
  return NextResponse.json(result);
}
```

**📖 More Patterns**: [`docs/development/component-patterns.md`](docs/development/component-patterns.md)

---

## 🔧 Common Tasks Quick Reference

### Add a New Feature (DDD Flow)

1. **Define Domain Entity/Value Object** in `src/domain/<context>/entities/`
2. **Create Use Case** in `src/application/<context>/use-cases/`
3. **Implement Repository** in `src/infrastructure/persistence/`
4. **Create UI/API** in `app/`

### Add shadcn/ui Component

```bash
npx shadcn@latest add button
```

```typescript
import { Button } from "@/app/components/ui/button";

<Button variant="default">Click me</Button>
```

### Format Code

```bash
npm run lint:fix  # Fix ESLint + Prettier formatting
```

**📖 Detailed Tasks**: [`docs/guides/common-tasks.md`](docs/guides/common-tasks.md)

---

## 🔄 Git Workflow

**Current Branch**: `claude/restructure-claude-docs-01Vb16zKZthN9UCXsHM75nDv`

### Commit Process

```bash
# Check status
git status

# Stage changes
git add .

# Commit with clear message
git commit -m "feat: add order management feature"

# Push to designated branch
git push -u origin claude/restructure-claude-docs-01Vb16zKZthN9UCXsHM75nDv
```

### Commit Message Format

- `feat:` - New feature
- `fix:` - Bug fix
- `refactor:` - Code restructuring
- `docs:` - Documentation changes
- `style:` - Code formatting
- `test:` - Test additions/changes

---

## 📝 AI Assistant Guidelines

### 🏛️ DDD Compliance Checklist

Before generating code, verify:

- [ ] **Correct Layer** - Is code in the right DDD layer?
- [ ] **No Layer Violations** - Respects dependency rules?
- [ ] **Business Logic Location** - In Domain layer, not UI/controllers?
- [ ] **Ubiquitous Language** - Naming matches business terminology?
- [ ] **Entity vs Value Object** - Correct pattern for the concept?
- [ ] **Repository Pattern** - Interface in Domain, implementation in Infrastructure?
- [ ] **No Anemic Models** - Entities contain behavior, not just getters/setters?

### When Making Changes

1. ✅ **Read files before editing** (use Read tool)
2. ✅ **Follow DDD layered architecture** strictly
3. ✅ **Match existing patterns**
4. ✅ **Use TypeScript strictly** (no `any`)
5. ✅ **Verify layer dependencies**
6. ✅ **Place business logic in Domain layer**
7. ✅ **Use Figma MCP to reference UI designs** when implementing components
8. ✅ **Run `npm run lint:fix` before committing**
9. ✅ **Test locally** with `npm run dev`

### UI Implementation Guidelines

- ✅ **Reference Figma designs via MCP** - Use Figma MCP to view and reference design specifications
- ✅ **Design-driven development** - Implement UI components based on Figma designs when available
- ✅ **Maintain design consistency** - Follow spacing, colors, typography from Figma specs
- ✅ **Ask for Figma access** - If design specs are needed but not accessible, request Figma file/link

### Code Generation Priorities

1. **Domain-First** - Start with Domain entities/value objects
2. **Ubiquitous Language** - Use business terminology
3. **Enforce Invariants** - Validate in constructors
4. **Pure Domain** - No framework dependencies in Domain layer
5. **Delegate to Domain** - Application/Presentation layers orchestrate, don't implement logic

**📖 Full AI Guidelines**: See original sections in detail docs

---

## 🐛 Quick Troubleshooting

| Issue                  | Solution                                                         |
| ---------------------- | ---------------------------------------------------------------- |
| Module not found       | `npm install`                                                    |
| TypeScript errors      | `npx tsc --noEmit`                                               |
| Tailwind not applying  | Restart dev server, check `@import "tailwindcss"` in globals.css |
| Port 3000 in use       | `npm run dev -- -p 3001`                                         |
| ESLint errors          | `npm run lint:fix`                                               |
| Hot reload not working | Clear `.next` folder, restart server                             |

**📖 Detailed Troubleshooting**: (To be created in guides)

---

## 📚 Resources

### Documentation

- [Next.js 16 Docs](https://nextjs.org/docs)
- [React 19 Docs](https://react.dev)
- [TypeScript Docs](https://www.typescriptlang.org/docs/)
- [Tailwind CSS v4](https://tailwindcss.com/docs)
- [shadcn/ui](https://ui.shadcn.com/)
- [Prettier](https://prettier.io/docs/)

### DDD Resources

- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
- [DDD Reference](https://www.domainlanguage.com/ddd/reference/)
- [Martin Fowler - DDD](https://martinfowler.com/tags/domain%20driven%20design.html)

---

## 📋 Documentation Changelog

### Version 3.2.0 (2025-11-17)

- ✅ **ADDED** - Frontend guidelines: All UI must be written in React
- ✅ **ADDED** - UI Implementation Guidelines section with Figma MCP usage
- ✅ **UPDATED** - AI Assistant guidelines to include Figma MCP for design reference

### Version 3.1.0 (2025-11-17)

- ✅ **ELIMINATED DUPLICATION** - Removed overlapping content between `specs/` and `docs/`
- ✅ **CONSOLIDATED** - Architecture and DDD details now in `specs/` only
- ✅ **REMOVED** `docs/architecture/` (duplicate of `specs/01-architecture.md`)
- ✅ **UPDATED** CLAUDE.md to reference `specs/` for project specifications
- ✅ **OPTIMIZED** context - Reduced redundancy for better AI performance
- ✅ **CLARIFIED** structure:
  - `specs/` → Project-specific specifications (architecture, domain model, etc.)
  - `docs/development/` → Next.js patterns and coding standards
  - `docs/guides/` → Common development tasks

### Version 3.0.0 (2025-11-17)

- ✅ **RESTRUCTURED** - Split documentation into modular files
- ✅ **REDUCED** main CLAUDE.md to ~440 lines (Quick Reference)
- ✅ **CREATED** `/docs` directory structure
- ✅ **SEPARATED** detailed guides into focused documents
- ✅ **IMPROVED** navigation with quick links

### Version 2.1.0 (2025-11-17)

- Added Prettier and shadcn/ui to tech stack

### Version 2.0.0 (2025-11-16)

- Added comprehensive DDD architecture

### Version 1.0.0 (2025-11-16)

- Initial CLAUDE.md creation

---

**Last Updated:** 2025-11-17
**Maintained By:** AI Assistants working on cafeteria-mg
**Version:** 3.2.0 - Frontend Guidelines Added

---

_For project specifications, see [`specs/`](specs/). For development guides, see [`docs/`](docs/)._
