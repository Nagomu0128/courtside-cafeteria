# CLAUDE.md - AI Assistant Guide for Cafeteria Management System

> **Last Updated:** 2025-11-17
> **Project:** Cafeteria Management System
> **Status:** Initial Development Phase

---

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture: Domain-Driven Design (DDD)](#architecture-domain-driven-design-ddd)
3. [Tech Stack](#tech-stack)
4. [Codebase Structure](#codebase-structure)
5. [Development Workflow](#development-workflow)
6. [Key Conventions](#key-conventions)
7. [File Organization Rules](#file-organization-rules)
8. [Coding Standards](#coding-standards)
9. [Component Patterns](#component-patterns)
10. [Styling Guidelines](#styling-guidelines)
11. [Common Tasks](#common-tasks)
12. [Deployment](#deployment)
13. [Troubleshooting](#troubleshooting)

---

## 🎯 Project Overview

**Cafeteria Management System** is a modern web application built with Next.js for managing cafeteria operations. The project is currently in its initial development phase with the base template setup.

**🏛️ CRITICAL: This project STRICTLY follows Domain-Driven Design (DDD) architecture principles. All implementation must adhere to DDD patterns and layered architecture.**

### Project Goals
- Provide an efficient cafeteria management solution
- **Implement clean architecture with DDD principles**
- **Maintain strict separation of concerns across DDD layers**
- Modern, responsive UI with dark mode support
- Type-safe development with TypeScript
- Fast, optimized production builds
- **Scalable and maintainable codebase following tactical DDD patterns**

### Architectural Principles
- ✅ **Domain-Driven Design (DDD)** - Core architectural pattern
- ✅ **Layered Architecture** - Separation of Domain, Application, Infrastructure, and Presentation
- ✅ **Ubiquitous Language** - Consistent terminology across code and business
- ✅ **Bounded Contexts** - Clear boundaries between domain modules
- ✅ **Aggregate Patterns** - Consistency boundaries in the domain model

### Current Status
- ✅ Base Next.js 16 template initialized
- ✅ TypeScript configuration complete
- ✅ Tailwind CSS v4 integrated
- ✅ shadcn/ui component library ready for use
- ✅ Prettier code formatter configured
- ✅ ESLint configured with Next.js rules
- 🚧 DDD directory structure to be implemented
- 🚧 Application features pending implementation

---

## 🛠 Tech Stack

### Core Framework
- **Next.js 16.0.3** - React framework with App Router
- **React 19.2.0** - Latest React with Server Components
- **TypeScript 5.x** - Type-safe JavaScript with strict mode

### Styling
- **Tailwind CSS v4** - Utility-first CSS framework (latest version)
- **shadcn/ui** - Re-usable components built with Radix UI and Tailwind CSS
- **PostCSS** - CSS processing with Tailwind plugin
- **Geist Font Family** - Vercel's optimized font (sans & mono)

### Development Tools
- **ESLint 9.x** - Linting with flat config format
- **Next.js ESLint Config** - Core Web Vitals rules
- **Prettier** - Opinionated code formatter for consistent code style
- **TypeScript Compiler** - Type checking (noEmit mode)

### Key Dependencies
```json
{
  "next": "16.0.3",
  "react": "19.2.0",
  "react-dom": "19.2.0",
  "typescript": "^5",
  "tailwindcss": "^4"
}
```

---

## 🏛️ Architecture: Domain-Driven Design (DDD)

### ⚠️ MANDATORY ARCHITECTURE COMPLIANCE

**This project STRICTLY adheres to Domain-Driven Design (DDD) principles. All code must follow DDD patterns and layered architecture. Non-compliance is not acceptable.**

### DDD Layered Architecture

The project follows a strict 4-layer architecture:

```
┌─────────────────────────────────────────┐
│   Presentation Layer (UI/API)          │  ← User Interface, Controllers, API Routes
├─────────────────────────────────────────┤
│   Application Layer (Use Cases)        │  ← Application Services, DTOs, Use Cases
├─────────────────────────────────────────┤
│   Domain Layer (Business Logic)        │  ← Entities, Value Objects, Domain Services
├─────────────────────────────────────────┤
│   Infrastructure Layer (External)      │  ← Repositories, DB, External APIs
└─────────────────────────────────────────┘
```

### Layer Responsibilities

#### 1. Domain Layer (Core Business Logic)
**Location:** `/src/domain/`

**Purpose:** Contains the core business logic and rules. This is the heart of the application.

**Components:**
- **Entities** - Objects with identity that persist over time
  ```typescript
  // src/domain/cafeteria/entities/Menu.ts
  export class Menu {
    constructor(
      private readonly id: MenuId,
      private name: MenuName,
      private price: Price,
      private availability: Availability
    ) {}
  }
  ```

- **Value Objects** - Immutable objects defined by their attributes
  ```typescript
  // src/domain/cafeteria/value-objects/Price.ts
  export class Price {
    constructor(private readonly amount: number) {
      if (amount < 0) throw new Error("Price cannot be negative");
    }
  }
  ```

- **Domain Services** - Business logic that doesn't belong to a single entity
  ```typescript
  // src/domain/cafeteria/services/OrderPricingService.ts
  export class OrderPricingService {
    calculateTotal(order: Order): Price {
      // Complex pricing logic
    }
  }
  ```

- **Aggregates** - Cluster of entities and value objects with a root entity
- **Domain Events** - Events that occurred in the domain
- **Repository Interfaces** - Contracts for data persistence (implementation in infrastructure)

**Rules:**
- ✅ Pure business logic only
- ✅ Framework-agnostic (no Next.js, React dependencies)
- ✅ No database or external service dependencies
- ❌ NEVER import from Application, Infrastructure, or Presentation layers
- ❌ NO side effects (I/O operations, HTTP calls, etc.)

#### 2. Application Layer (Use Cases)
**Location:** `/src/application/`

**Purpose:** Orchestrates domain objects to perform application-specific tasks.

**Components:**
- **Use Cases / Application Services** - Implement specific application operations
  ```typescript
  // src/application/cafeteria/use-cases/CreateOrderUseCase.ts
  export class CreateOrderUseCase {
    constructor(
      private orderRepository: IOrderRepository,
      private menuRepository: IMenuRepository
    ) {}

    async execute(dto: CreateOrderDto): Promise<OrderDto> {
      // Orchestrate domain objects
      const menu = await this.menuRepository.findById(dto.menuId);
      const order = Order.create(menu, dto.quantity);
      await this.orderRepository.save(order);
      return OrderDto.fromEntity(order);
    }
  }
  ```

- **DTOs (Data Transfer Objects)** - Data contracts for communication
- **Application Services** - Coordinate multiple use cases
- **Command/Query Handlers** - CQRS pattern implementation

**Rules:**
- ✅ Can import from Domain layer
- ✅ Depends on Repository interfaces (not implementations)
- ✅ Orchestrates domain logic
- ❌ NO business logic (delegate to Domain layer)
- ❌ NO direct database access (use repositories)
- ❌ NEVER import from Infrastructure or Presentation layers

#### 3. Infrastructure Layer (External Concerns)
**Location:** `/src/infrastructure/`

**Purpose:** Implements technical capabilities (persistence, external APIs, etc.).

**Components:**
- **Repository Implementations** - Concrete implementations of domain repositories
  ```typescript
  // src/infrastructure/persistence/prisma/repositories/PrismaOrderRepository.ts
  export class PrismaOrderRepository implements IOrderRepository {
    async save(order: Order): Promise<void> {
      await prisma.order.create({
        data: this.toSchema(order)
      });
    }
  }
  ```

- **Database Access** - ORM models, queries
- **External Services** - Third-party API clients
- **Frameworks** - Database adapters, messaging systems

**Rules:**
- ✅ Can import from Domain and Application layers
- ✅ Implements interfaces defined in Domain layer
- ✅ Contains all I/O operations
- ❌ NO business logic
- ❌ NEVER import from Presentation layer

#### 4. Presentation Layer (UI/API)
**Location:** `/app/` (Next.js App Router)

**Purpose:** Handles user interaction and displays information.

**Components:**
- **Pages** - Next.js page components
- **Components** - React UI components
- **API Routes** - Next.js API endpoints
- **Controllers** - Handle HTTP requests/responses

**Rules:**
- ✅ Can import from Application and Infrastructure layers (via DI)
- ✅ Handles HTTP requests/responses
- ✅ Renders UI
- ❌ NO business logic (delegate to Application layer)
- ❌ NO direct Domain layer access (use Application layer)

### DDD Tactical Patterns

#### Entities
- Have unique identity
- Mutable state
- Equality by ID, not by attributes

```typescript
export class Order {
  constructor(
    private readonly id: OrderId,
    private status: OrderStatus,
    private items: OrderItem[]
  ) {}

  addItem(item: OrderItem): void {
    // Business rule enforcement
    if (this.status === OrderStatus.COMPLETED) {
      throw new Error("Cannot modify completed order");
    }
    this.items.push(item);
  }
}
```

#### Value Objects
- No identity
- Immutable
- Equality by attributes

```typescript
export class Email {
  private readonly value: string;

  constructor(email: string) {
    if (!this.isValid(email)) {
      throw new Error("Invalid email format");
    }
    this.value = email;
  }

  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  getValue(): string {
    return this.value;
  }
}
```

#### Aggregates
- Cluster of entities and value objects
- Aggregate Root controls access
- Transactional consistency boundary

```typescript
export class Order { // Aggregate Root
  private items: OrderItem[]; // Part of aggregate

  addItem(menuId: MenuId, quantity: number): void {
    // Enforce invariants
    const item = new OrderItem(menuId, quantity);
    this.items.push(item);
    this.recalculateTotal();
  }
}
```

#### Domain Services
- Operations that don't naturally fit in entities/value objects
- Stateless

```typescript
export class OrderPricingService {
  calculateDiscount(order: Order, customer: Customer): Price {
    // Complex logic involving multiple entities
  }
}
```

#### Repository Pattern
- Interface in Domain, Implementation in Infrastructure
- Collection-like interface for aggregates

```typescript
// Domain layer
export interface IOrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
  delete(id: OrderId): Promise<void>;
}

// Infrastructure layer
export class PrismaOrderRepository implements IOrderRepository {
  async findById(id: OrderId): Promise<Order | null> {
    const data = await prisma.order.findUnique({
      where: { id: id.getValue() }
    });
    return data ? this.toDomain(data) : null;
  }
}
```

#### Domain Events
- Something that happened in the domain
- Past tense naming

```typescript
export class OrderPlacedEvent {
  constructor(
    public readonly orderId: OrderId,
    public readonly customerId: CustomerId,
    public readonly occurredAt: Date
  ) {}
}
```

### Dependency Rule

**CRITICAL: Dependencies flow inward only**

```
Presentation → Application → Domain
Infrastructure → Application → Domain
                   ↑
            (interfaces only)
```

- **Domain Layer** - No dependencies on other layers
- **Application Layer** - Depends only on Domain
- **Infrastructure Layer** - Depends on Domain and Application (implements interfaces)
- **Presentation Layer** - Depends on Application and Infrastructure (via DI)

### Bounded Contexts

Organize code by business domains:

```
src/domain/
├── cafeteria/          # Cafeteria Context
│   ├── entities/
│   ├── value-objects/
│   ├── services/
│   └── repositories/
├── inventory/          # Inventory Context
├── billing/            # Billing Context
└── shared/             # Shared Kernel
```

### Ubiquitous Language

- Use business terminology in code
- Class/method names match business concepts
- Collaborate with domain experts for naming

**Good:**
```typescript
class Menu {
  markAsUnavailable(): void { }
}
```

**Bad:**
```typescript
class Menu {
  setFlag(value: boolean): void { }
}
```

---

## 📁 Codebase Structure

**🏛️ This structure strictly follows DDD (Domain-Driven Design) layered architecture.**

```
cafeteria-mg/
├── src/                          # Source code (DDD layers)
│   ├── domain/                  # DOMAIN LAYER (Core Business Logic)
│   │   ├── cafeteria/           # Cafeteria Bounded Context
│   │   │   ├── entities/        # Domain Entities
│   │   │   │   ├── Menu.ts
│   │   │   │   ├── Order.ts
│   │   │   │   └── Customer.ts
│   │   │   ├── value-objects/   # Value Objects
│   │   │   │   ├── MenuId.ts
│   │   │   │   ├── Price.ts
│   │   │   │   ├── Email.ts
│   │   │   │   └── OrderStatus.ts
│   │   │   ├── services/        # Domain Services
│   │   │   │   └── OrderPricingService.ts
│   │   │   ├── repositories/    # Repository Interfaces
│   │   │   │   ├── IMenuRepository.ts
│   │   │   │   └── IOrderRepository.ts
│   │   │   ├── events/          # Domain Events
│   │   │   │   └── OrderPlacedEvent.ts
│   │   │   └── exceptions/      # Domain Exceptions
│   │   │       └── InvalidOrderException.ts
│   │   ├── inventory/           # Inventory Bounded Context
│   │   ├── billing/             # Billing Bounded Context
│   │   └── shared/              # Shared Kernel (cross-context)
│   │       ├── value-objects/
│   │       └── interfaces/
│   │
│   ├── application/             # APPLICATION LAYER (Use Cases)
│   │   ├── cafeteria/           # Cafeteria Use Cases
│   │   │   ├── use-cases/       # Use Case implementations
│   │   │   │   ├── CreateOrderUseCase.ts
│   │   │   │   ├── UpdateMenuUseCase.ts
│   │   │   │   └── GetMenuListUseCase.ts
│   │   │   ├── dtos/            # Data Transfer Objects
│   │   │   │   ├── CreateOrderDto.ts
│   │   │   │   ├── OrderDto.ts
│   │   │   │   └── MenuDto.ts
│   │   │   ├── services/        # Application Services
│   │   │   │   └── OrderApplicationService.ts
│   │   │   └── ports/           # Port interfaces (Hexagonal)
│   │   │       ├── input/
│   │   │       └── output/
│   │   └── shared/              # Shared Application concerns
│   │       └── interfaces/
│   │
│   ├── infrastructure/          # INFRASTRUCTURE LAYER (External)
│   │   ├── persistence/         # Database implementations
│   │   │   ├── prisma/          # Prisma ORM
│   │   │   │   ├── schema.prisma
│   │   │   │   ├── repositories/
│   │   │   │   │   ├── PrismaMenuRepository.ts
│   │   │   │   │   └── PrismaOrderRepository.ts
│   │   │   │   └── migrations/
│   │   │   └── typeorm/         # Alternative: TypeORM
│   │   ├── external-services/   # External API clients
│   │   │   ├── payment/
│   │   │   └── notification/
│   │   ├── messaging/           # Event bus, message queue
│   │   └── config/              # Infrastructure config
│   │       └── database.config.ts
│   │
│   └── presentation/            # PRESENTATION LAYER (partial)
│       └── api/                 # API-specific presentation logic
│           └── controllers/     # API Controllers
│
├── app/                         # PRESENTATION LAYER (Next.js App Router)
│   ├── (routes)/                # Route groups
│   │   ├── (public)/            # Public routes
│   │   │   ├── page.tsx
│   │   │   └── menu/
│   │   │       └── page.tsx
│   │   ├── (dashboard)/         # Dashboard routes
│   │   │   ├── layout.tsx
│   │   │   ├── orders/
│   │   │   │   └── page.tsx
│   │   │   └── inventory/
│   │   │       └── page.tsx
│   │   └── (auth)/              # Auth routes
│   │       ├── login/
│   │       └── register/
│   ├── api/                     # API Routes (Next.js)
│   │   ├── orders/
│   │   │   └── route.ts
│   │   └── menu/
│   │       └── route.ts
│   ├── components/              # Presentation Components
│   │   ├── ui/                  # shadcn/ui components (auto-generated)
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   └── dialog.tsx
│   │   ├── features/            # Feature-specific components
│   │   │   ├── orders/
│   │   │   └── menu/
│   │   ├── shared/              # Shared custom components
│   │   │   └── CustomComponent.tsx
│   │   └── layouts/             # Layout components
│   ├── layout.tsx               # Root layout
│   ├── page.tsx                 # Home page
│   └── globals.css              # Global styles
│
├── public/                      # Static assets
│   ├── images/
│   └── icons/
│
├── tests/                       # Test files (mirror src structure)
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── Configuration Files
├── .gitignore
├── eslint.config.mjs
├── next.config.ts
├── package.json
├── postcss.config.mjs
├── tsconfig.json
├── README.md
└── CLAUDE.md                    # This file
```

### Directory Purposes (DDD Compliant)

| Directory/File | Layer | Purpose | Rules |
|---------------|-------|---------|-------|
| `/src/domain/` | **Domain** | Core business logic, entities, value objects | ✅ Framework-agnostic<br>❌ NO external dependencies |
| `/src/application/` | **Application** | Use cases, orchestration, DTOs | ✅ Import from Domain<br>❌ NO UI or DB code |
| `/src/infrastructure/` | **Infrastructure** | DB, external APIs, I/O operations | ✅ Implements Domain interfaces<br>❌ NO business logic |
| `/app/` | **Presentation** | Next.js routes, pages, UI components | ✅ Import from Application<br>❌ NO direct Domain access |
| `/src/presentation/api/` | **Presentation** | API controllers (if needed) | ✅ Delegates to Application layer |
| `/public/` | Static | Static assets served at root | Public files only |
| `/tests/` | Testing | Unit, integration, E2E tests | Mirror src/ structure |

### Layer Dependencies (Enforced)

```
┌─────────────────────────────────────────────────┐
│  app/ (Presentation)                            │
│  - Can import: application/, infrastructure/    │
│  - Cannot import: domain/ directly              │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  src/application/ (Application)                 │
│  - Can import: domain/                          │
│  - Cannot import: infrastructure/, app/         │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  src/domain/ (Domain)                           │
│  - Can import: NOTHING (pure business logic)    │
│  - Cannot import: ANY other layer               │
└─────────────────────────────────────────────────┘
                    ↑
┌─────────────────────────────────────────────────┐
│  src/infrastructure/ (Infrastructure)           │
│  - Can import: domain/, application/            │
│  - Cannot import: app/                          │
└─────────────────────────────────────────────────┘
```

### Bounded Context Organization

Each bounded context (e.g., `cafeteria/`, `inventory/`, `billing/`) follows the same structure:

```
<context-name>/
├── entities/           # Domain entities
├── value-objects/      # Value objects
├── services/           # Domain services
├── repositories/       # Repository interfaces
├── events/             # Domain events
└── exceptions/         # Domain-specific exceptions
```

---

## 🔄 Development Workflow

### Initial Setup

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Development server runs at http://localhost:3000
```

### Available Scripts

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `npm run dev` | Start dev server with hot reload | Active development |
| `npm run build` | Create production build | Before deployment, testing |
| `npm run start` | Start production server | After build, production testing |
| `npm run lint` | Run ESLint checks | Before commits, CI/CD |
| `npm run lint:fix` | Fix ESLint errors and format code with Prettier | Before commits, code cleanup |

### Git Workflow

**Current Branch:** `claude/claude-md-mi1sbqajkfshs7p1-01UqNYd2mVvLGwMHd4xi1CsW`

**Branch Naming Convention:**
- Feature branches: `claude/claude-md-*`
- Always push to designated branch with `-u origin` flag

**Commit Guidelines:**
1. Make focused, atomic commits
2. Use clear, descriptive commit messages
3. Run lint before committing: `npm run lint`
4. Ensure TypeScript compiles without errors

**Example Commit Flow:**
```bash
# After making changes
git add .
git commit -m "feat: add user authentication component"
git push -u origin claude/claude-md-mi1sbqajkfshs7p1-01UqNYd2mVvLGwMHd4xi1CsW
```

---

## 🎨 Key Conventions

### TypeScript Conventions

1. **Strict Mode Enabled** - All strict TypeScript checks active
2. **Type Imports** - Use `import type` for type-only imports
   ```typescript
   import type { Metadata } from "next";
   ```

3. **Explicit Return Types** - Prefer explicit return types for functions
   ```typescript
   export default function Home(): JSX.Element {
     return <div>...</div>;
   }
   ```

4. **No Implicit Any** - Always define types, never rely on `any`

### React Conventions

1. **Server Components by Default** - Components are Server Components unless marked with `"use client"`
2. **Default Exports** - Pages and layouts use default exports
3. **Named Props Interfaces** - Define props with TypeScript interfaces
   ```typescript
   interface ButtonProps {
     label: string;
     onClick: () => void;
   }
   ```

4. **Readonly Props** - Use `Readonly<>` for component props
   ```typescript
   export default function Layout({ children }: Readonly<{ children: React.ReactNode }>) {
     // ...
   }
   ```

### File Naming Conventions

| File Type | Convention | Example |
|-----------|-----------|---------|
| Pages | `page.tsx` | `app/dashboard/page.tsx` |
| Layouts | `layout.tsx` | `app/dashboard/layout.tsx` |
| Components | PascalCase | `UserProfile.tsx` |
| Utilities | camelCase | `formatDate.ts` |
| Types | PascalCase | `User.types.ts` |
| Constants | SCREAMING_SNAKE_CASE | `API_ENDPOINTS.ts` |

### Import Path Conventions

- Use `@/` alias for root-relative imports
  ```typescript
  import { Button } from "@/components/Button";
  import { formatDate } from "@/utils/formatDate";
  ```

---

## 📂 File Organization Rules

**🏛️ All organization follows DDD layered architecture principles.**

### Domain Layer Organization

**Location:** `/src/domain/<bounded-context>/`

**Structure:**
```
domain/cafeteria/
├── entities/              # Domain Entities
│   ├── Menu.ts
│   ├── Order.ts
│   └── index.ts
├── value-objects/         # Value Objects
│   ├── Price.ts
│   ├── MenuId.ts
│   └── index.ts
├── services/              # Domain Services
│   └── OrderPricingService.ts
├── repositories/          # Repository Interfaces ONLY
│   ├── IMenuRepository.ts
│   └── IOrderRepository.ts
├── events/                # Domain Events
│   └── OrderPlacedEvent.ts
└── exceptions/            # Domain Exceptions
    └── InvalidOrderException.ts
```

**Naming Conventions:**
- **Entities**: PascalCase, business nouns (e.g., `Order`, `Menu`)
- **Value Objects**: PascalCase, descriptive nouns (e.g., `Price`, `Email`)
- **Repository Interfaces**: `I<Entity>Repository` (e.g., `IOrderRepository`)
- **Domain Services**: `<Action><Entity>Service` (e.g., `OrderPricingService`)
- **Domain Events**: Past tense (e.g., `OrderPlacedEvent`, `MenuUpdatedEvent`)

**Rules:**
- ✅ Only business logic
- ✅ Framework-agnostic TypeScript
- ✅ Use Ubiquitous Language from business domain
- ❌ NO imports from other layers
- ❌ NO I/O operations
- ❌ NO framework dependencies (React, Next.js, etc.)

### Application Layer Organization

**Location:** `/src/application/<bounded-context>/`

**Structure:**
```
application/cafeteria/
├── use-cases/             # Use Case implementations
│   ├── CreateOrderUseCase.ts
│   ├── UpdateMenuUseCase.ts
│   └── index.ts
├── dtos/                  # Data Transfer Objects
│   ├── CreateOrderDto.ts
│   ├── OrderDto.ts
│   └── index.ts
├── services/              # Application Services
│   └── OrderApplicationService.ts
└── ports/                 # Hexagonal Architecture Ports
    ├── input/
    └── output/
```

**Naming Conventions:**
- **Use Cases**: `<Verb><Entity>UseCase` (e.g., `CreateOrderUseCase`)
- **DTOs**: `<Purpose>Dto` (e.g., `CreateOrderDto`, `OrderDto`)
- **Application Services**: `<Entity>ApplicationService`

**Rules:**
- ✅ Orchestrate domain objects
- ✅ Import from Domain layer only
- ✅ Define DTOs for data transfer
- ❌ NO business logic (delegate to Domain)
- ❌ NO direct database access
- ❌ NO imports from Infrastructure or Presentation

### Infrastructure Layer Organization

**Location:** `/src/infrastructure/`

**Structure:**
```
infrastructure/
├── persistence/
│   ├── prisma/
│   │   ├── schema.prisma
│   │   ├── repositories/
│   │   │   ├── PrismaOrderRepository.ts
│   │   │   └── PrismaMenuRepository.ts
│   │   └── migrations/
│   └── typeorm/
├── external-services/
│   ├── payment/
│   │   └── StripePaymentService.ts
│   └── notification/
│       └── EmailNotificationService.ts
├── messaging/
│   └── EventBus.ts
└── config/
    └── database.config.ts
```

**Naming Conventions:**
- **Repository Implementations**: `<Technology><Entity>Repository` (e.g., `PrismaOrderRepository`)
- **External Services**: `<Provider><Service>Service` (e.g., `StripePaymentService`)

**Rules:**
- ✅ Implement Domain repository interfaces
- ✅ All I/O operations
- ✅ Framework/library specific code
- ❌ NO business logic
- ❌ NO imports from Presentation layer

### Presentation Layer Organization (Next.js)

**Location:** `/app/`

**Structure:**
```
app/
├── (routes)/              # Route groups (DDD contexts)
│   ├── (public)/
│   │   ├── page.tsx
│   │   └── menu/
│   │       └── page.tsx
│   ├── (dashboard)/
│   │   ├── layout.tsx
│   │   └── orders/
│   │       ├── page.tsx
│   │       └── components/
│   │           └── OrderList.tsx
│   └── (auth)/
│       ├── login/
│       └── register/
├── api/                   # API Routes
│   ├── orders/
│   │   └── route.ts
│   └── menu/
│       └── route.ts
├── components/
│   ├── features/          # Feature components (by context)
│   │   ├── orders/
│   │   │   ├── OrderCard.tsx
│   │   │   └── OrderForm.tsx
│   │   └── menu/
│   │       └── MenuList.tsx
│   ├── shared/            # Shared UI components
│   │   ├── Button.tsx
│   │   ├── Card.tsx
│   │   └── Modal.tsx
│   └── layouts/
│       └── DashboardLayout.tsx
├── layout.tsx
├── page.tsx
└── globals.css
```

**Component Organization:**

1. **shadcn/ui Components** - Auto-generated UI primitives
   ```
   app/components/ui/
   ├── button.tsx          # Button component with variants
   ├── card.tsx            # Card component with sub-components
   ├── dialog.tsx          # Dialog/Modal component
   ├── input.tsx           # Input component
   └── ...                 # Other shadcn/ui components
   ```
   - Generated via `npx shadcn@latest add <component>`
   - Built with Radix UI and Tailwind CSS
   - Fully customizable and type-safe
   - Follow lowercase naming convention (shadcn/ui standard)

2. **Feature Components** - Organize by Bounded Context
   ```
   app/components/features/orders/
   ├── OrderCard.tsx
   ├── OrderForm.tsx
   └── OrderList.tsx
   ```

3. **Shared Custom Components** - Reusable custom components
   ```
   app/components/shared/
   ├── CustomComponent.tsx
   └── AnotherComponent.tsx
   ```
   - For custom components not from shadcn/ui
   - Build on top of shadcn/ui components when possible

4. **Route-Specific Components** - Co-locate with route
   ```
   app/(dashboard)/orders/
   ├── page.tsx
   └── components/
       └── OrderDashboard.tsx
   ```

**API Routes (Controllers):**
```typescript
// app/api/orders/route.ts
import { CreateOrderUseCase } from "@/src/application/cafeteria/use-cases/CreateOrderUseCase";
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  const useCase = new CreateOrderUseCase(/* inject dependencies */);
  const dto = await request.json();
  const result = await useCase.execute(dto);
  return NextResponse.json(result);
}
```

**Rules:**
- ✅ Import from Application layer (use cases)
- ✅ Inject Infrastructure implementations via DI
- ✅ Handle HTTP/UI concerns only
- ❌ NO business logic
- ❌ NO direct Domain layer imports
- ❌ NO direct database access

### Route Organization (App Router with DDD)

```
app/
├── (public)/             # Public Context (no auth)
│   ├── page.tsx         # /
│   └── menu/
│       └── page.tsx     # /menu
├── (dashboard)/          # Dashboard Context (auth required)
│   ├── layout.tsx       # Dashboard layout
│   ├── orders/
│   │   ├── page.tsx     # /orders
│   │   └── [id]/
│   │       └── page.tsx # /orders/[id]
│   └── inventory/
│       └── page.tsx     # /inventory
├── (auth)/              # Auth Context
│   ├── login/
│   │   └── page.tsx    # /login
│   └── register/
│       └── page.tsx    # /register
└── api/                 # API Routes
    ├── orders/
    │   ├── route.ts    # /api/orders
    │   └── [id]/
    │       └── route.ts # /api/orders/[id]
    └── menu/
        └── route.ts     # /api/menu
```

### CSS Organization

1. **Global Styles** - Keep in `app/globals.css`
2. **Component Styles** - Use Tailwind classes inline
3. **Custom CSS Variables** - Define in globals.css
   ```css
   :root {
     --custom-color: #value;
   }
   ```

### Import Path Organization

**Use path aliases:**
```typescript
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./*"],
      "@/domain/*": ["./src/domain/*"],
      "@/application/*": ["./src/application/*"],
      "@/infrastructure/*": ["./src/infrastructure/*"]
    }
  }
}
```

**Import Examples:**
```typescript
// ✅ Correct: Presentation imports Application
import { CreateOrderUseCase } from "@/application/cafeteria/use-cases/CreateOrderUseCase";

// ✅ Correct: Application imports Domain
import { Order } from "@/domain/cafeteria/entities/Order";

// ✅ Correct: Infrastructure imports Domain interface
import { IOrderRepository } from "@/domain/cafeteria/repositories/IOrderRepository";

// ❌ Wrong: Application importing Infrastructure
import { PrismaOrderRepository } from "@/infrastructure/persistence/prisma/repositories/PrismaOrderRepository";

// ❌ Wrong: Presentation importing Domain directly
import { Order } from "@/domain/cafeteria/entities/Order";
```

---

## 💻 Coding Standards

**🏛️ All code must follow DDD principles and layered architecture rules.**

### DDD Architecture Standards

✅ **MANDATORY DDD RULES:**

1. **Respect Layer Boundaries**
   - Domain layer NEVER imports from other layers
   - Application layer ONLY imports from Domain
   - Infrastructure implements Domain interfaces
   - Presentation uses Application layer (not Domain directly)

2. **Use Ubiquitous Language**
   - Match business terminology in code
   - Class/method names reflect business concepts
   - Consistent naming across layers

3. **Enforce Invariants in Entities**
   - Business rules in entity methods
   - Validate in constructors
   - No anemic domain models

4. **Immutable Value Objects**
   - No setters in value objects
   - Validate in constructor
   - Equality by value, not reference

5. **Repository Pattern**
   - Interface in Domain layer
   - Implementation in Infrastructure
   - Return domain entities, not DB models

6. **Dependency Injection**
   - Inject dependencies via constructor
   - Depend on interfaces, not implementations

❌ **PROHIBITED PRACTICES:**
- ❌ Anemic domain models (entities with only getters/setters)
- ❌ Business logic in controllers or API routes
- ❌ Domain layer depending on Infrastructure
- ❌ Direct database access from Application layer
- ❌ Presentation layer importing Domain entities directly
- ❌ Breaking layer dependency rules

### TypeScript Standards

✅ **DO:**
- Enable all strict mode checks
- Use explicit types for function parameters
- Define interfaces for object shapes
- Use `unknown` instead of `any` when type is truly unknown
- Leverage type inference where obvious
- **Use classes for Entities and Value Objects (DDD)**
- **Use interfaces for Repository contracts (DDD)**

❌ **DON'T:**
- Use `any` type (use `unknown` or proper types)
- Disable TypeScript errors with `@ts-ignore`
- Use non-null assertions (`!`) without justification
- Leave unused variables or imports
- **Use plain objects for domain entities**
- **Put business logic in DTOs or plain objects**

### React Standards

✅ **DO:**
- Use Server Components by default (no "use client" needed)
- Add "use client" only when needed (hooks, event handlers, browser APIs)
- Destructure props in function parameters
- Use semantic HTML elements
- Keep components focused and single-purpose
- Export metadata from pages/layouts for SEO

❌ **DON'T:**
- Use `any` for props types
- Mutate props directly
- Create deeply nested component trees
- Mix business logic with presentation
- Forget to add `key` prop in lists

### Tailwind CSS Standards

✅ **DO:**
- Use Tailwind utility classes for styling
- Follow mobile-first responsive design (`sm:`, `md:`, `lg:`)
- Use dark mode classes (`dark:`)
- Leverage CSS variables from globals.css
- Keep class lists readable (break into multiple lines if needed)

❌ **DON'T:**
- Write inline `style={}` unless absolutely necessary
- Create custom CSS when Tailwind utility exists
- Use arbitrary values excessively (e.g., `w-[347px]`)
- Ignore responsive design considerations

### Code Quality Standards

1. **DRY Principle** - Don't Repeat Yourself
2. **KISS Principle** - Keep It Simple, Stupid
3. **Consistent Formatting** - Let Prettier and ESLint handle it
4. **Clear Naming** - Use descriptive, unambiguous names
5. **Comments** - Explain "why", not "what"

### Prettier Standards

✅ **DO:**
- Run Prettier before committing code
- Use `npm run lint:fix` to auto-fix linting issues and format all files
- Follow Prettier's default configuration (no custom overrides unless necessary)
- Let Prettier handle code formatting (indentation, line breaks, quotes, etc.)
- Integrate Prettier with your IDE for format-on-save

❌ **DON'T:**
- Manually format code when Prettier can handle it
- Commit unformatted code
- Disable Prettier rules without team discussion
- Mix formatted and unformatted code in the same PR

---

## 🧩 Component Patterns

### Server Component Pattern (Default)

```typescript
// app/users/page.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Users",
  description: "User management page",
};

export default async function UsersPage() {
  // Can directly fetch data here
  const users = await fetchUsers();

  return (
    <div>
      <h1>Users</h1>
      {users.map((user) => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  );
}
```

### Client Component Pattern

```typescript
// app/components/Counter.tsx
"use client";

import { useState } from "react";

interface CounterProps {
  initialCount?: number;
}

export default function Counter({ initialCount = 0 }: Readonly<CounterProps>) {
  const [count, setCount] = useState(initialCount);

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

### Layout Pattern

```typescript
// app/dashboard/layout.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: {
    template: "%s | Dashboard",
    default: "Dashboard",
  },
};

export default function DashboardLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <div className="flex min-h-screen">
      <aside>Sidebar</aside>
      <main className="flex-1">{children}</main>
    </div>
  );
}
```

### shadcn/ui Component Pattern

**shadcn/ui** components are located in `app/components/ui/` and are built with Radix UI primitives and Tailwind CSS.

**Adding a shadcn/ui component:**
```bash
# Add a specific component (e.g., button)
npx shadcn@latest add button

# Add multiple components
npx shadcn@latest add button card dialog
```

**Using shadcn/ui components:**
```typescript
// app/components/features/orders/OrderCard.tsx
import { Button } from "@/app/components/ui/button";
import { Card, CardContent, CardDescription, CardFooter, CardHeader, CardTitle } from "@/app/components/ui/card";

export default function OrderCard({ order }: { order: Order }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{order.title}</CardTitle>
        <CardDescription>{order.description}</CardDescription>
      </CardHeader>
      <CardContent>
        <p>Total: ${order.total}</p>
      </CardContent>
      <CardFooter>
        <Button onClick={() => handleOrder(order.id)}>
          Place Order
        </Button>
      </CardFooter>
    </Card>
  );
}
```

**Customizing shadcn/ui components:**
- Components are copied to your project, so you can modify them directly
- Maintain consistent styling with Tailwind CSS
- Components are fully type-safe with TypeScript
- Follow DDD principles: use in Presentation layer only

---

## 🎨 Styling Guidelines

### Tailwind CSS v4 Setup

The project uses Tailwind CSS v4 with the new import syntax:

```css
/* app/globals.css */
@import "tailwindcss";
```

### Color Scheme

**Current CSS Variables:**
```css
:root {
  --background: #ffffff;
  --foreground: #171717;
}

@media (prefers-color-scheme: dark) {
  :root {
    --background: #0a0a0a;
    --foreground: #ededed;
  }
}
```

### Dark Mode Implementation

Use Tailwind's `dark:` variant:
```typescript
<div className="bg-white dark:bg-black text-zinc-900 dark:text-zinc-50">
  Content
</div>
```

### Responsive Design

Follow mobile-first approach:
```typescript
<div className="w-full md:w-1/2 lg:w-1/3">
  // Full width on mobile, half on tablet, third on desktop
</div>
```

### Common Utility Patterns

```typescript
// Flexbox centering
className="flex items-center justify-center"

// Grid layout
className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4"

// Spacing
className="px-4 py-2"  // padding
className="mx-auto"     // horizontal center
className="space-y-4"   // vertical spacing between children

// Typography
className="text-lg font-medium leading-8"

// Interactive states
className="hover:bg-gray-100 dark:hover:bg-gray-800 transition-colors"
```

---

## 🔧 Common Tasks

### Adding a New Page

1. Create page file in app directory:
   ```typescript
   // app/about/page.tsx
   export default function AboutPage() {
     return <div>About Us</div>;
   }
   ```

2. Add metadata:
   ```typescript
   import type { Metadata } from "next";

   export const metadata: Metadata = {
     title: "About",
     description: "About our cafeteria",
   };
   ```

3. Access at `/about`

### Adding a Component

1. Create component file:
   ```typescript
   // app/components/Button.tsx
   interface ButtonProps {
     label: string;
     onClick: () => void;
   }

   export default function Button({ label, onClick }: Readonly<ButtonProps>) {
     return (
       <button onClick={onClick} className="px-4 py-2 bg-blue-500 text-white rounded">
         {label}
       </button>
     );
   }
   ```

2. Import and use:
   ```typescript
   import Button from "@/app/components/Button";

   <Button label="Click me" onClick={() => console.log("Clicked")} />
   ```

### Adding an API Route

1. Create route handler:
   ```typescript
   // app/api/users/route.ts
   import { NextResponse } from "next/server";

   export async function GET() {
     const users = [{ id: 1, name: "John" }];
     return NextResponse.json(users);
   }
   ```

2. Access at `/api/users`

### Adding Global CSS Variables

1. Edit `app/globals.css`:
   ```css
   :root {
     --primary-color: #3b82f6;
     --secondary-color: #8b5cf6;
   }
   ```

2. Use in Tailwind:
   ```typescript
   <div className="bg-[var(--primary-color)]">Content</div>
   ```

### Updating Metadata

```typescript
// Static metadata
export const metadata: Metadata = {
  title: "Page Title",
  description: "Page description",
  keywords: ["keyword1", "keyword2"],
};

// Dynamic metadata
export async function generateMetadata({ params }): Promise<Metadata> {
  return {
    title: `User ${params.id}`,
  };
}
```

### Adding shadcn/ui Components

1. **Install a component:**
   ```bash
   npx shadcn@latest add button
   ```

2. **Use the component:**
   ```typescript
   import { Button } from "@/app/components/ui/button";

   <Button variant="default">Click me</Button>
   <Button variant="destructive">Delete</Button>
   <Button variant="outline">Cancel</Button>
   ```

3. **Available variants and sizes:**
   - Variants: `default`, `destructive`, `outline`, `secondary`, `ghost`, `link`
   - Sizes: `default`, `sm`, `lg`, `icon`

4. **Customize in `app/components/ui/button.tsx`** if needed

### Formatting Code with Prettier

1. **Fix linting issues and format all files:**
   ```bash
   npm run lint:fix
   ```
   This command runs ESLint with `--fix` flag and formats code with Prettier.

2. **Format specific file or directory manually:**
   ```bash
   npx prettier --write app/components/Button.tsx
   npx prettier --write "app/**/*.{ts,tsx}"
   ```

3. **IDE Integration:**
   - Install Prettier extension for your IDE
   - Enable "Format on Save" in settings
   - Prettier will auto-format on file save

---

## 🚀 Deployment

### Vercel Deployment (Recommended)

1. **Connect Repository**
   - Import project to Vercel
   - Auto-detects Next.js configuration

2. **Environment Variables**
   - Set in Vercel dashboard
   - Access via `process.env.VARIABLE_NAME`

3. **Deploy**
   - Auto-deploys on git push to main branch
   - Preview deployments for PRs

### Build Checklist

Before deploying:

- ✅ Run `npm run build` successfully
- ✅ Run `npm run lint` with no errors
- ✅ Test production build locally: `npm run start`
- ✅ Check for TypeScript errors
- ✅ Verify all environment variables set
- ✅ Test in multiple browsers
- ✅ Verify responsive design

### Performance Optimization

Next.js automatically optimizes:
- ✅ Image optimization via `next/image`
- ✅ Font optimization via `next/font`
- ✅ Code splitting and lazy loading
- ✅ Static page generation where possible

---

## 🐛 Troubleshooting

### Common Issues

#### "Module not found" errors
```bash
# Solution: Install dependencies
npm install
```

#### TypeScript errors after changes
```bash
# Solution: Check tsconfig.json and run type check
npx tsc --noEmit
```

#### Tailwind classes not applying
```bash
# Check globals.css has @import "tailwindcss"
# Restart dev server
npm run dev
```

#### Port 3000 already in use
```bash
# Run on different port
npm run dev -- -p 3001
```

### Build Errors

#### "Type error: Cannot find module"
- Check import paths
- Verify `@/` alias in tsconfig.json
- Ensure file extensions are correct (.tsx vs .ts)

#### ESLint errors blocking build
```bash
# Fix auto-fixable issues
npm run lint -- --fix

# View detailed error report
npm run lint
```

### Development Tips

1. **Hot Reload Not Working**
   - Restart dev server
   - Check file is in `app/` directory
   - Clear `.next` folder: `rm -rf .next`

2. **Slow Build Times**
   - Clear `.next` folder
   - Update dependencies: `npm update`
   - Check for large files in public/

3. **CSS Not Updating**
   - Hard refresh browser (Cmd+Shift+R / Ctrl+Shift+R)
   - Clear browser cache
   - Restart dev server

---

## 📝 Notes for AI Assistants

**⚠️ CRITICAL: This project uses Domain-Driven Design (DDD) architecture. All code generation and modifications MUST follow DDD principles.**

### 🏛️ DDD Implementation Checklist

Before generating or modifying code, verify:

- [ ] **Correct Layer** - Is the code in the right layer?
- [ ] **No Layer Violations** - Does it respect dependency rules?
- [ ] **Business Logic Location** - Is business logic in Domain layer?
- [ ] **Ubiquitous Language** - Does naming match business terminology?
- [ ] **Entity vs Value Object** - Correct pattern for the concept?
- [ ] **Repository Pattern** - Interface in Domain, implementation in Infrastructure?
- [ ] **No Anemic Models** - Do entities contain behavior, not just data?

### When Making Changes

1. **Always read files before editing** - Use Read tool first
2. **Follow DDD layered architecture** - Respect layer boundaries strictly
3. **Follow existing patterns** - Match the codebase style
4. **Use TypeScript strictly** - No `any` types
5. **Verify layer dependencies** - Check imports don't violate DDD rules
6. **Place business logic in Domain layer** - Never in controllers or UI
7. **Test locally** - Run `npm run dev` to verify changes
8. **Run lint** - Execute `npm run lint` before committing
9. **Commit atomically** - One logical change per commit

### Code Generation Guidelines

**DDD-Specific Guidelines:**

1. **Domain Layer (Business Logic)**
   ```typescript
   // ✅ Good: Entity with behavior
   export class Order {
     addItem(item: OrderItem): void {
       if (this.isCompleted()) {
         throw new Error("Cannot modify completed order");
       }
       this.items.push(item);
     }
   }

   // ❌ Bad: Anemic entity
   export class Order {
     getItems(): OrderItem[] { return this.items; }
     setItems(items: OrderItem[]): void { this.items = items; }
   }
   ```

2. **Application Layer (Use Cases)**
   ```typescript
   // ✅ Good: Use case orchestrating domain
   export class CreateOrderUseCase {
     async execute(dto: CreateOrderDto): Promise<OrderDto> {
       const menu = await this.menuRepository.findById(dto.menuId);
       const order = Order.create(menu, dto.quantity); // Domain logic
       await this.orderRepository.save(order);
       return OrderDto.fromEntity(order);
     }
   }

   // ❌ Bad: Business logic in use case
   export class CreateOrderUseCase {
     async execute(dto: CreateOrderDto): Promise<OrderDto> {
       if (dto.quantity < 0) { // Should be in domain
         throw new Error("Invalid quantity");
       }
     }
   }
   ```

3. **Infrastructure Layer (Implementation)**
   ```typescript
   // ✅ Good: Implements domain interface
   export class PrismaOrderRepository implements IOrderRepository {
     async save(order: Order): Promise<void> {
       await prisma.order.create({
         data: this.toSchema(order)
       });
     }
   }
   ```

4. **Presentation Layer (UI/API)**
   ```typescript
   // ✅ Good: Delegates to application layer
   export async function POST(request: NextRequest) {
     const useCase = new CreateOrderUseCase(orderRepo);
     const dto = await request.json();
     return NextResponse.json(await useCase.execute(dto));
   }

   // ❌ Bad: Business logic in controller
   export async function POST(request: NextRequest) {
     const data = await request.json();
     if (data.price < 0) { // Should be in domain
       return NextResponse.json({ error: "Invalid" });
     }
   }
   ```

**General Guidelines:**

5. **Components**: Use functional components with TypeScript
6. **Styling**: Prefer Tailwind utilities over custom CSS
7. **Server vs Client**: Default to Server Components, use "use client" only when necessary
8. **Imports**: Use `@/` alias for clean imports
9. **Metadata**: Always export metadata for pages
10. **Error Handling**: Add proper error boundaries and fallbacks

### Best Practices for AI Collaboration

**DDD-Specific Practices:**
- **Identify the Bounded Context** first before creating files
- **Determine if it's an Entity or Value Object** before implementing
- **Define repository interfaces in Domain** before implementing in Infrastructure
- **Use Ubiquitous Language** - ask for business terminology if unclear
- **Keep Domain layer pure** - no framework dependencies
- **Enforce invariants in constructors** - validate domain rules immediately

**General Practices:**
- **Ask for clarification** if requirements are ambiguous
- **Suggest alternatives** when better patterns exist
- **Document complex logic** with clear comments
- **Follow existing conventions** rather than introducing new patterns
- **Consider performance** - Server Components over Client when possible
- **Think about accessibility** - semantic HTML, ARIA labels
- **Mobile-first design** - responsive by default

### What Not to Do

**DDD Violations (CRITICAL):**
- ❌ **NEVER** put business logic in controllers or API routes
- ❌ **NEVER** import Domain entities directly in Presentation layer
- ❌ **NEVER** access database directly from Application layer
- ❌ **NEVER** create anemic domain models (just getters/setters)
- ❌ **NEVER** violate layer dependency rules
- ❌ **NEVER** use framework-specific code in Domain layer
- ❌ **NEVER** implement repositories in Application layer

**General Violations:**
- Don't create files in root directory unnecessarily
- Don't add dependencies without discussing
- Don't disable TypeScript checks
- Don't ignore ESLint errors
- Don't write custom CSS when Tailwind utility exists
- Don't commit `node_modules`, `.next`, or `.env` files

### DDD Code Review Questions

When reviewing generated code, ask:

1. **Is this business logic?** → Should be in Domain layer
2. **Is this orchestration?** → Should be in Application layer
3. **Is this I/O or external service?** → Should be in Infrastructure layer
4. **Is this UI or API handling?** → Should be in Presentation layer
5. **Does it violate dependency rules?** → Fix the imports
6. **Is the entity anemic?** → Add behavior methods
7. **Is validation in the right place?** → Domain entities/value objects
8. **Are we using Ubiquitous Language?** → Match business terminology

---

## 📚 Additional Resources

### Official Documentation

- [Next.js 16 Docs](https://nextjs.org/docs)
- [React 19 Docs](https://react.dev)
- [TypeScript Docs](https://www.typescriptlang.org/docs/)
- [Tailwind CSS v4 Docs](https://tailwindcss.com/docs)
- [shadcn/ui Docs](https://ui.shadcn.com/) - Component library and documentation
- [Prettier Docs](https://prettier.io/docs/en/) - Code formatter documentation

### Domain-Driven Design Resources

**Essential Reading:**
- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/) - The original DDD book
- [Implementing Domain-Driven Design by Vaughn Vernon](https://vaughnvernon.com/) - Practical DDD implementation
- [DDD Reference by Eric Evans](https://www.domainlanguage.com/ddd/reference/) - Quick reference guide

**Online Resources:**
- [Martin Fowler - DDD](https://martinfowler.com/tags/domain%20driven%20design.html) - Articles on DDD patterns
- [Microsoft DDD Guide](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/) - DDD in .NET (concepts apply to TypeScript)
- [DDD Community](https://www.dddcommunity.org/) - DDD community resources

**Key Concepts:**
- **Entities** - Objects with unique identity
- **Value Objects** - Immutable objects defined by attributes
- **Aggregates** - Consistency boundaries
- **Repositories** - Abstraction for data access
- **Domain Services** - Stateless operations
- **Domain Events** - Something that happened in the domain
- **Bounded Contexts** - Explicit boundaries
- **Ubiquitous Language** - Common vocabulary

### Learning Resources

- [Next.js Learn](https://nextjs.org/learn) - Interactive tutorial
- [React Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)
- [App Router Migration](https://nextjs.org/docs/app/building-your-application/upgrading/app-router-migration)
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) - Architectural principles complementing DDD

---

## 🔄 Maintenance

### Updating Dependencies

```bash
# Check for updates
npm outdated

# Update specific package
npm update next

# Update all packages (carefully)
npm update
```

### Regular Maintenance Tasks

- Weekly: Check for security vulnerabilities (`npm audit`)
- Monthly: Update dependencies
- Quarterly: Review and remove unused code
- As needed: Update this CLAUDE.md file

---

**Last Updated:** 2025-11-17
**Maintained By:** AI Assistants working on cafeteria-mg
**Version:** 2.1.0 - Prettier and shadcn/ui Added

---

*This document should be updated whenever significant architectural decisions are made or new patterns are established.*

## 📋 Document Changelog

### Version 2.1.0 (2025-11-17)
- ✅ Added **Prettier** code formatter to Tech Stack
- ✅ Added **shadcn/ui** component library to Tech Stack
- ✅ Updated Available Scripts with `npm run lint:fix` command (combines ESLint fix + Prettier)
- ✅ Added Prettier Standards section to Coding Standards
- ✅ Added shadcn/ui Component Pattern section
- ✅ Updated Component Organization to include `app/components/ui/` directory
- ✅ Added Common Tasks for shadcn/ui components and Prettier formatting
- ✅ Updated Codebase Structure to reflect shadcn/ui component directory
- ✅ Added shadcn/ui and Prettier documentation links to Additional Resources
- ✅ Updated Current Status to reflect new tooling

### Version 2.0.0 (2025-11-16)
- ✅ Added comprehensive Domain-Driven Design (DDD) architecture section
- ✅ Updated codebase structure to follow DDD layered architecture
- ✅ Added DDD file organization rules for all layers
- ✅ Included DDD tactical patterns (Entities, Value Objects, Aggregates, etc.)
- ✅ Added DDD coding standards and best practices
- ✅ Updated AI assistant guidelines with DDD compliance rules
- ✅ Added DDD learning resources and references
- ✅ Enforced strict layer dependency rules
- ⚠️ **BREAKING CHANGE**: All future code must follow DDD architecture

### Version 1.0.0 (2025-11-16)
- Initial CLAUDE.md creation
- Basic Next.js 16 project documentation
- Tech stack and codebase structure
- Development workflow and conventions
