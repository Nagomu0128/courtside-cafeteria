# Coding Standards

> **Version:** 2.0.0
> **Last Updated:** 2025-11-17

---

## Table of Contents

1. [Overview](#overview)
2. [DDD Architecture Standards](#ddd-architecture-standards)
3. [TypeScript Standards](#typescript-standards)
4. [React Standards](#react-standards)
5. [Tailwind CSS Standards](#tailwind-css-standards)
6. [Code Quality Standards](#code-quality-standards)
7. [Prettier Standards](#prettier-standards)
8. [Testing Standards](#testing-standards)

---

## Overview

**All code must follow DDD principles and layered architecture rules.**

This document defines coding standards for Next.js development in the Cafeteria Management System project.

**📖 For DDD architecture details, see:** [`specs/01-architecture.md`](../../specs/01-architecture.md)

---

## DDD Architecture Standards

**⚠️ For complete DDD architecture specifications, see [`specs/01-architecture.md`](../../specs/01-architecture.md)**

This section provides Next.js-specific DDD implementation guidelines. For foundational DDD concepts and project architecture, refer to the specs.

### Mandatory DDD Rules

#### 1. Respect Layer Boundaries

✅ **DO:**

- Domain layer NEVER imports from other layers
- Application layer ONLY imports from Domain
- Infrastructure implements Domain interfaces
- Presentation uses Application layer (not Domain directly)

❌ **DON'T:**

- Import Application/Infrastructure/Presentation in Domain
- Import Infrastructure/Presentation in Application
- Import Presentation in Infrastructure
- Import Domain directly in Presentation

**Example:**

```typescript
// ✅ Good: Application imports Domain
// src/application/cafeteria/use-cases/CreateOrderUseCase.ts
import { Order } from "@/src/domain/cafeteria/entities/Order";
import { IOrderRepository } from "@/src/domain/cafeteria/repositories/IOrderRepository";

export class CreateOrderUseCase {
  constructor(private orderRepository: IOrderRepository) {}
}

// ❌ Bad: Domain imports Application
// src/domain/cafeteria/entities/Order.ts
import { CreateOrderDto } from "@/src/application/cafeteria/dtos/CreateOrderDto"; // WRONG!
```

#### 2. Use Ubiquitous Language

✅ **DO:**

- Match business terminology in code
- Class/method names reflect business concepts
- Consistent naming across layers
- Document business rules with comments

❌ **DON'T:**

- Use technical jargon in domain layer
- Generic names like `process()`, `handle()`, `doSomething()`
- Abbreviations unless universally understood

**Example:**

```typescript
// ✅ Good: Clear business terminology
class Menu {
  markAsUnavailable(): void {
    this.availability = Availability.UNAVAILABLE;
  }

  updatePrice(newPrice: Price): void {
    if (!newPrice.isValid()) {
      throw new InvalidPriceException();
    }
    this.price = newPrice;
  }
}

// ❌ Bad: Technical/vague terminology
class Menu {
  setFlag(value: boolean): void {
    // What flag?
    this.flag = value;
  }

  update(data: any): void {
    // Too generic
    this.data = data;
  }
}
```

#### 3. Enforce Invariants in Entities

✅ **DO:**

- Business rules in entity methods
- Validate in constructors (fail fast)
- Maintain consistency in aggregates
- Throw domain exceptions for rule violations

❌ **DON'T:**

- Allow invalid state to exist
- Validate outside of entities
- Use setters without validation
- Create anemic domain models

**Example:**

```typescript
// ✅ Good: Invariants enforced
export class Order {
  constructor(
    private readonly id: OrderId,
    private status: OrderStatus,
    private items: OrderItem[]
  ) {
    if (items.length === 0) {
      throw new InvalidOrderException("Order must have at least one item");
    }
  }

  addItem(item: OrderItem): void {
    if (this.status === OrderStatus.COMPLETED) {
      throw new InvalidOrderException("Cannot modify completed order");
    }
    this.items.push(item);
    this.recalculateTotal();
  }
}

// ❌ Bad: No invariant enforcement
export class Order {
  public status: string;
  public items: any[];

  setStatus(status: string): void {
    this.status = status; // No validation
  }

  setItems(items: any[]): void {
    this.items = items; // No validation
  }
}
```

#### 4. Immutable Value Objects

✅ **DO:**

- No setters in value objects
- Validate in constructor
- Equality by value, not reference
- Make fields readonly

❌ **DON'T:**

- Allow mutation after creation
- Use identity comparison
- Expose internal state

**Example:**

```typescript
// ✅ Good: Immutable value object
export class Email {
  private readonly value: string;

  constructor(email: string) {
    if (!this.isValid(email)) {
      throw new InvalidEmailException("Invalid email format");
    }
    this.value = email;
  }

  getValue(): string {
    return this.value;
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }

  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}

// ❌ Bad: Mutable value object
export class Email {
  public value: string; // Not readonly, can be mutated

  setValue(value: string): void {
    // Setter allows mutation
    this.value = value;
  }
}
```

#### 5. Repository Pattern

✅ **DO:**

- Interface in Domain layer
- Implementation in Infrastructure layer
- Return domain entities, not DB models
- Collection-like interface

❌ **DON'T:**

- Implement repositories in Application layer
- Return database-specific types
- Expose ORM details to domain

**Example:**

```typescript
// ✅ Good: Domain interface, Infrastructure implementation

// Domain layer
export interface IOrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

// Infrastructure layer
export class PrismaOrderRepository implements IOrderRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: OrderId): Promise<Order | null> {
    const data = await this.prisma.order.findUnique({
      where: { id: id.getValue() },
    });
    return data ? this.toDomain(data) : null;
  }

  private toDomain(data: any): Order {
    // Map database model to domain entity
  }
}

// ❌ Bad: Repository in Application layer
// src/application/cafeteria/repositories/OrderRepository.ts
export class OrderRepository {
  // WRONG LAYER!
  async findById(id: string): Promise<any> {
    return await prisma.order.findUnique({ where: { id } });
  }
}
```

#### 6. Dependency Injection

✅ **DO:**

- Inject dependencies via constructor
- Depend on interfaces, not implementations
- Use DI in Presentation layer

❌ **DON'T:**

- Create dependencies inside classes
- Depend on concrete implementations
- Use global state

**Example:**

```typescript
// ✅ Good: Constructor injection with interfaces
export class CreateOrderUseCase {
  constructor(
    private orderRepository: IOrderRepository, // Interface
    private menuRepository: IMenuRepository // Interface
  ) {}

  async execute(dto: CreateOrderDto): Promise<OrderDto> {
    const menu = await this.menuRepository.findById(dto.menuId);
    // ...
  }
}

// Presentation layer - Inject concrete implementations
const useCase = new CreateOrderUseCase(
  new PrismaOrderRepository(prisma), // Concrete implementation
  new PrismaMenuRepository(prisma)
);

// ❌ Bad: Creating dependencies inside
export class CreateOrderUseCase {
  async execute(dto: CreateOrderDto): Promise<OrderDto> {
    const orderRepository = new PrismaOrderRepository(); // WRONG!
    const menu = await orderRepository.findById(dto.menuId);
  }
}
```

### Prohibited Practices

❌ **NEVER:**

- Anemic domain models (entities with only getters/setters)
- Business logic in controllers or API routes
- Domain layer depending on Infrastructure
- Direct database access from Application layer
- Presentation layer importing Domain entities directly
- Breaking layer dependency rules
- Using framework-specific code in Domain layer
- Implementing repositories in Application layer

---

## TypeScript Standards

### General Rules

✅ **DO:**

- Enable all strict mode checks
- Use explicit types for function parameters
- Define interfaces for object shapes
- Use `unknown` instead of `any` when type is truly unknown
- Leverage type inference where obvious
- Use classes for Entities and Value Objects (DDD)
- Use interfaces for Repository contracts (DDD)

❌ **DON'T:**

- Use `any` type (use `unknown` or proper types)
- Disable TypeScript errors with `@ts-ignore`
- Use non-null assertions (`!`) without justification
- Leave unused variables or imports
- Use plain objects for domain entities
- Put business logic in DTOs or plain objects

### Type Definitions

```typescript
// ✅ Good: Explicit types
function calculateTotal(price: number, quantity: number): number {
  return price * quantity;
}

interface OrderProps {
  id: string;
  total: number;
  status: string;
}

type OrderStatus = 'pending' | 'completed' | 'cancelled';

// ❌ Bad: Implicit any
function calculate(a, b) {  // Implicit any
  return a * b;
}

const order: any = { ... };  // any type
```

### Interfaces vs Types

```typescript
// ✅ Use interface for object shapes
interface User {
  id: string;
  name: string;
  email: string;
}

// ✅ Use type for unions, intersections, primitives
type OrderStatus = "pending" | "completed" | "cancelled";
type ID = string | number;
type UserWithTimestamps = User & { createdAt: Date; updatedAt: Date };

// ❌ Don't use type for simple object shapes
type User = {
  // Prefer interface
  id: string;
  name: string;
};
```

### Null Safety

```typescript
// ✅ Good: Handle null/undefined explicitly
function getUser(id: string): User | null {
  const user = users.find((u) => u.id === id);
  return user ?? null;
}

const user = getUser("123");
if (user !== null) {
  console.log(user.name);
}

// ❌ Bad: Non-null assertion without check
const user = getUser("123");
console.log(user!.name); // Dangerous!
```

### Type Imports

```typescript
// ✅ Good: Use type imports for type-only imports
import type { Metadata } from "next";
import type { User } from "./types";

// Runtime import
import { Button } from "./Button";

// ❌ Bad: Regular import for types
import { Metadata } from "next"; // Runtime import for type
```

---

## React Standards

### Component Types

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

### Server Components

```typescript
// ✅ Good: Server Component (default)
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Orders',
  description: 'Order management page'
};

export default async function OrdersPage() {
  // Can fetch data directly
  const orders = await fetchOrders();

  return (
    <div>
      <h1>Orders</h1>
      {orders.map(order => (
        <OrderCard key={order.id} order={order} />
      ))}
    </div>
  );
}
```

### Client Components

```typescript
// ✅ Good: Client Component with "use client"
'use client';

import { useState } from 'react';

interface CounterProps {
  initialCount?: number;
}

export function Counter({ initialCount = 0 }: Readonly<CounterProps>) {
  const [count, setCount] = useState(initialCount);

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}

// ❌ Bad: Client Component without "use client"
import { useState } from 'react';  // ERROR: Can't use hooks in Server Component

export function Counter() {
  const [count, setCount] = useState(0);  // Will fail
  return <button>Count: {count}</button>;
}
```

### Props Interfaces

```typescript
// ✅ Good: Typed props with Readonly
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary';
  disabled?: boolean;
}

export function Button({
  label,
  onClick,
  variant = 'primary',
  disabled = false
}: Readonly<ButtonProps>) {
  return (
    <button onClick={onClick} disabled={disabled}>
      {label}
    </button>
  );
}

// ❌ Bad: any props
export function Button(props: any) {  // No type safety
  return <button>{props.label}</button>;
}
```

### Key Props in Lists

```typescript
// ✅ Good: Unique key prop
{orders.map(order => (
  <OrderCard key={order.id} order={order} />
))}

// ❌ Bad: Index as key (can cause issues)
{orders.map((order, index) => (
  <OrderCard key={index} order={order} />
))}
```

---

## Tailwind CSS Standards

### General Rules

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

### Responsive Design

```typescript
// ✅ Good: Mobile-first responsive
<div className="w-full md:w-1/2 lg:w-1/3">
  {/* Full width on mobile, half on tablet, third on desktop */}
</div>

<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  {/* 1 column mobile, 2 tablet, 3 desktop */}
</div>

// ❌ Bad: Not responsive
<div className="w-1/3">
  {/* Fixed width on all screens */}
</div>
```

### Dark Mode

```typescript
// ✅ Good: Dark mode support
<div className="bg-white dark:bg-black text-zinc-900 dark:text-zinc-50">
  Content
</div>

<button className="bg-blue-500 hover:bg-blue-600 dark:bg-blue-700 dark:hover:bg-blue-800">
  Button
</button>
```

### Class Organization

```typescript
// ✅ Good: Organized, readable classes
<div
  className="
    flex items-center justify-between
    px-4 py-2
    bg-white dark:bg-gray-800
    rounded-lg shadow-md
    hover:shadow-lg transition-shadow
  "
>
  {/* Content */}
</div>

// ❌ Bad: Long, unreadable class string
<div className="flex items-center justify-between px-4 py-2 bg-white dark:bg-gray-800 rounded-lg shadow-md hover:shadow-lg transition-shadow">
  {/* Content */}
</div>
```

### Common Patterns

```typescript
// Flexbox centering
className = "flex items-center justify-center";

// Grid layout
className = "grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4";

// Spacing
className = "px-4 py-2"; // Padding
className = "mx-auto"; // Horizontal center
className = "space-y-4"; // Vertical spacing between children

// Typography
className = "text-lg font-medium leading-8";

// Interactive states
className = "hover:bg-gray-100 dark:hover:bg-gray-800 transition-colors";
```

---

## Code Quality Standards

### Principles

1. **DRY (Don't Repeat Yourself)**
   - Extract repeated code into functions/components
   - Use composition over duplication

2. **KISS (Keep It Simple, Stupid)**
   - Prefer simple solutions over complex ones
   - Break complex functions into smaller ones

3. **SOLID Principles**
   - Single Responsibility
   - Open/Closed
   - Liskov Substitution
   - Interface Segregation
   - Dependency Inversion

4. **Code Formatting**
   - Let Prettier and ESLint handle it
   - Run `npm run lint:fix` before committing

5. **Clear Naming**
   - Use descriptive, unambiguous names
   - Follow naming conventions

6. **Comments**
   - Explain "why", not "what"
   - Document business rules
   - Use JSDoc for public APIs

### Function Length

```typescript
// ✅ Good: Short, focused functions
function calculateTotal(items: OrderItem[]): number {
  return items.reduce((sum, item) => sum + item.getSubtotal(), 0);
}

function validateEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

// ❌ Bad: Long, complex function
function processOrder(data: any): any {
  // 100+ lines of code doing multiple things
  // Validation, calculation, persistence, notification, etc.
}
```

### Comments

```typescript
// ✅ Good: Explain why, document business rules
export class Order {
  addItem(item: OrderItem): void {
    // Business rule: Cannot modify completed orders
    // This ensures order integrity after completion
    if (this.status === OrderStatus.COMPLETED) {
      throw new InvalidOrderException("Cannot modify completed order");
    }
    this.items.push(item);
  }
}

// ❌ Bad: Explain what (code is self-explanatory)
export class Order {
  addItem(item: OrderItem): void {
    // Check if status is completed
    if (this.status === OrderStatus.COMPLETED) {
      // Throw exception
      throw new InvalidOrderException("Cannot modify completed order");
    }
    // Add item to items array
    this.items.push(item);
  }
}
```

---

## Prettier Standards

### General Rules

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

### Commands

```bash
# Format and fix all files
npm run lint:fix

# Format specific file
npx prettier --write app/components/Button.tsx

# Format directory
npx prettier --write "app/**/*.{ts,tsx}"

# Check formatting (CI)
npx prettier --check "**/*.{ts,tsx}"
```

### IDE Integration

**VS Code:**

1. Install "Prettier - Code formatter" extension
2. Enable "Format on Save" in settings
3. Set Prettier as default formatter

**Settings.json:**

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

---

## Testing Standards

### Test Organization

```
tests/
├── unit/                  # Unit tests (Domain, Application)
│   ├── domain/
│   │   └── cafeteria/
│   │       ├── entities/
│   │       │   └── Order.test.ts
│   │       └── value-objects/
│   │           └── Price.test.ts
│   └── application/
│       └── cafeteria/
│           └── use-cases/
│               └── CreateOrderUseCase.test.ts
├── integration/           # Integration tests (Infrastructure)
│   └── persistence/
│       └── PrismaOrderRepository.test.ts
└── e2e/                  # End-to-end tests (Presentation)
    └── orders/
        └── create-order.test.ts
```

### Unit Test Example

```typescript
// tests/unit/domain/cafeteria/entities/Order.test.ts
import { describe, it, expect } from "vitest";
import { Order } from "@/src/domain/cafeteria/entities/Order";
import { OrderId } from "@/src/domain/cafeteria/value-objects/OrderId";
import { InvalidOrderException } from "@/src/domain/cafeteria/exceptions/InvalidOrderException";

describe("Order", () => {
  it("should create a valid order", () => {
    const order = new Order(new OrderId("123"), OrderStatus.PENDING, []);

    expect(order.getId().getValue()).toBe("123");
  });

  it("should throw exception when adding item to completed order", () => {
    const order = new Order(new OrderId("123"), OrderStatus.COMPLETED, []);

    expect(() => {
      order.addItem(new OrderItem());
    }).toThrow(InvalidOrderException);
  });
});
```

---

## Summary

### Key Takeaways

1. **DDD Compliance** - Follow layer boundaries strictly
2. **TypeScript** - Use strict mode, explicit types, no `any`
3. **React** - Server Components by default, "use client" only when needed
4. **Tailwind** - Mobile-first, dark mode support, readable classes
5. **Code Quality** - DRY, KISS, SOLID principles
6. **Prettier** - Auto-format before committing
7. **Testing** - Unit tests for domain, integration for infrastructure

### Quick Checklist

Before committing:

- [ ] Code follows DDD layer rules
- [ ] No layer dependency violations
- [ ] TypeScript strict mode passes
- [ ] `npm run lint:fix` executed
- [ ] Tests pass
- [ ] Business logic in Domain layer
- [ ] Ubiquitous language used

---

**Back to:** [CLAUDE.md](../../CLAUDE.md)
