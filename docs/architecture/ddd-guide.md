# Domain-Driven Design (DDD) Architecture Guide

> **Version:** 1.0.0
> **Last Updated:** 2025-11-17

---

## Table of Contents

1. [Overview](#overview)
2. [DDD Layered Architecture](#ddd-layered-architecture)
3. [Layer Responsibilities](#layer-responsibilities)
4. [DDD Tactical Patterns](#ddd-tactical-patterns)
5. [Dependency Rule](#dependency-rule)
6. [Bounded Contexts](#bounded-contexts)
7. [Ubiquitous Language](#ubiquitous-language)
8. [Best Practices](#best-practices)

---

## Overview

**⚠️ CRITICAL:** This project STRICTLY adheres to Domain-Driven Design (DDD) principles. All code must follow DDD patterns and layered architecture. Non-compliance is not acceptable.

### Why DDD?

- **Complexity Management**: Handle complex business logic in a structured way
- **Maintainability**: Clear separation of concerns across layers
- **Scalability**: Modular architecture that grows with your needs
- **Ubiquitous Language**: Consistent terminology between code and business

### Architectural Principles

- ✅ **Domain-Driven Design (DDD)** - Core architectural pattern
- ✅ **Layered Architecture** - Separation of Domain, Application, Infrastructure, and Presentation
- ✅ **Ubiquitous Language** - Consistent terminology across code and business
- ✅ **Bounded Contexts** - Clear boundaries between domain modules
- ✅ **Aggregate Patterns** - Consistency boundaries in the domain model

---

## DDD Layered Architecture

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

---

## Layer Responsibilities

### 1. Domain Layer (Core Business Logic)

**Location:** `/src/domain/`

**Purpose:** Contains the core business logic and rules. This is the heart of the application.

#### Components

##### Entities
Objects with identity that persist over time.

```typescript
// src/domain/cafeteria/entities/Menu.ts
export class Menu {
  constructor(
    private readonly id: MenuId,
    private name: MenuName,
    private price: Price,
    private availability: Availability
  ) {}

  updatePrice(newPrice: Price): void {
    if (newPrice.isNegative()) {
      throw new InvalidPriceException("Price cannot be negative");
    }
    this.price = newPrice;
  }

  markAsUnavailable(): void {
    this.availability = Availability.UNAVAILABLE;
  }
}
```

##### Value Objects
Immutable objects defined by their attributes.

```typescript
// src/domain/cafeteria/value-objects/Price.ts
export class Price {
  private readonly amount: number;

  constructor(amount: number) {
    if (amount < 0) {
      throw new Error("Price cannot be negative");
    }
    this.amount = amount;
  }

  isNegative(): boolean {
    return this.amount < 0;
  }

  getValue(): number {
    return this.amount;
  }

  equals(other: Price): boolean {
    return this.amount === other.amount;
  }
}
```

##### Domain Services
Business logic that doesn't belong to a single entity.

```typescript
// src/domain/cafeteria/services/OrderPricingService.ts
export class OrderPricingService {
  calculateTotal(order: Order): Price {
    let total = 0;
    for (const item of order.getItems()) {
      total += item.getPrice().getValue() * item.getQuantity();
    }
    return new Price(total);
  }

  calculateDiscount(order: Order, customer: Customer): Price {
    // Complex pricing logic involving multiple entities
    if (customer.isPremium()) {
      return new Price(this.calculateTotal(order).getValue() * 0.9);
    }
    return this.calculateTotal(order);
  }
}
```

##### Repository Interfaces
Contracts for data persistence (implementation in infrastructure).

```typescript
// src/domain/cafeteria/repositories/IOrderRepository.ts
export interface IOrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findByCustomerId(customerId: CustomerId): Promise<Order[]>;
  save(order: Order): Promise<void>;
  delete(id: OrderId): Promise<void>;
}
```

#### Rules

- ✅ Pure business logic only
- ✅ Framework-agnostic (no Next.js, React dependencies)
- ✅ No database or external service dependencies
- ❌ NEVER import from Application, Infrastructure, or Presentation layers
- ❌ NO side effects (I/O operations, HTTP calls, etc.)

---

### 2. Application Layer (Use Cases)

**Location:** `/src/application/`

**Purpose:** Orchestrates domain objects to perform application-specific tasks.

#### Components

##### Use Cases / Application Services
Implement specific application operations.

```typescript
// src/application/cafeteria/use-cases/CreateOrderUseCase.ts
export class CreateOrderUseCase {
  constructor(
    private orderRepository: IOrderRepository,
    private menuRepository: IMenuRepository,
    private pricingService: OrderPricingService
  ) {}

  async execute(dto: CreateOrderDto): Promise<OrderDto> {
    // 1. Validate input
    if (!dto.menuId || !dto.quantity) {
      throw new ValidationException("Invalid input");
    }

    // 2. Load domain objects
    const menu = await this.menuRepository.findById(new MenuId(dto.menuId));
    if (!menu) {
      throw new NotFoundException("Menu not found");
    }

    // 3. Create domain entity (business logic)
    const order = Order.create(menu, dto.quantity);

    // 4. Calculate total using domain service
    const total = this.pricingService.calculateTotal(order);
    order.setTotal(total);

    // 5. Persist
    await this.orderRepository.save(order);

    // 6. Return DTO
    return OrderDto.fromEntity(order);
  }
}
```

##### DTOs (Data Transfer Objects)
Data contracts for communication between layers.

```typescript
// src/application/cafeteria/dtos/CreateOrderDto.ts
export class CreateOrderDto {
  constructor(
    public readonly menuId: string,
    public readonly quantity: number,
    public readonly customerId: string
  ) {}
}

// src/application/cafeteria/dtos/OrderDto.ts
export class OrderDto {
  constructor(
    public readonly id: string,
    public readonly total: number,
    public readonly status: string,
    public readonly createdAt: Date
  ) {}

  static fromEntity(order: Order): OrderDto {
    return new OrderDto(
      order.getId().getValue(),
      order.getTotal().getValue(),
      order.getStatus().toString(),
      order.getCreatedAt()
    );
  }
}
```

#### Rules

- ✅ Can import from Domain layer
- ✅ Depends on Repository interfaces (not implementations)
- ✅ Orchestrates domain logic
- ❌ NO business logic (delegate to Domain layer)
- ❌ NO direct database access (use repositories)
- ❌ NEVER import from Infrastructure or Presentation layers

---

### 3. Infrastructure Layer (External Concerns)

**Location:** `/src/infrastructure/`

**Purpose:** Implements technical capabilities (persistence, external APIs, etc.).

#### Components

##### Repository Implementations
Concrete implementations of domain repositories.

```typescript
// src/infrastructure/persistence/prisma/repositories/PrismaOrderRepository.ts
import { PrismaClient } from '@prisma/client';
import { IOrderRepository } from '@/src/domain/cafeteria/repositories/IOrderRepository';
import { Order } from '@/src/domain/cafeteria/entities/Order';

export class PrismaOrderRepository implements IOrderRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: OrderId): Promise<Order | null> {
    const data = await this.prisma.order.findUnique({
      where: { id: id.getValue() },
      include: { items: true }
    });

    return data ? this.toDomain(data) : null;
  }

  async save(order: Order): Promise<void> {
    const data = this.toSchema(order);

    await this.prisma.order.upsert({
      where: { id: data.id },
      create: data,
      update: data
    });
  }

  async delete(id: OrderId): Promise<void> {
    await this.prisma.order.delete({
      where: { id: id.getValue() }
    });
  }

  // Mapper: Domain → Database Schema
  private toSchema(order: Order): any {
    return {
      id: order.getId().getValue(),
      total: order.getTotal().getValue(),
      status: order.getStatus().toString(),
      createdAt: order.getCreatedAt()
    };
  }

  // Mapper: Database Schema → Domain
  private toDomain(data: any): Order {
    return new Order(
      new OrderId(data.id),
      new OrderStatus(data.status),
      data.items.map(item => this.mapOrderItem(item)),
      new Price(data.total),
      data.createdAt
    );
  }
}
```

##### External Services
Third-party API clients.

```typescript
// src/infrastructure/external-services/payment/StripePaymentService.ts
export class StripePaymentService {
  constructor(private stripeClient: Stripe) {}

  async processPayment(amount: Price, customerId: CustomerId): Promise<PaymentResult> {
    const intent = await this.stripeClient.paymentIntents.create({
      amount: amount.getValue() * 100, // Convert to cents
      currency: 'usd',
      customer: customerId.getValue()
    });

    return new PaymentResult(intent.id, intent.status);
  }
}
```

#### Rules

- ✅ Can import from Domain and Application layers
- ✅ Implements interfaces defined in Domain layer
- ✅ Contains all I/O operations
- ❌ NO business logic
- ❌ NEVER import from Presentation layer

---

### 4. Presentation Layer (UI/API)

**Location:** `/app/` (Next.js App Router)

**Purpose:** Handles user interaction and displays information.

#### Components

##### API Routes (Controllers)
Handle HTTP requests/responses.

```typescript
// app/api/orders/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { CreateOrderUseCase } from '@/src/application/cafeteria/use-cases/CreateOrderUseCase';
import { CreateOrderDto } from '@/src/application/cafeteria/dtos/CreateOrderDto';

export async function POST(request: NextRequest) {
  try {
    // 1. Parse request
    const body = await request.json();

    // 2. Create DTO
    const dto = new CreateOrderDto(
      body.menuId,
      body.quantity,
      body.customerId
    );

    // 3. Inject dependencies (DI)
    const useCase = new CreateOrderUseCase(
      orderRepository,  // Infrastructure
      menuRepository,   // Infrastructure
      pricingService    // Domain
    );

    // 4. Execute use case
    const result = await useCase.execute(dto);

    // 5. Return response
    return NextResponse.json(result, { status: 201 });

  } catch (error) {
    return NextResponse.json(
      { error: error.message },
      { status: 400 }
    );
  }
}
```

##### React Components
UI components for rendering.

```typescript
// app/components/features/orders/OrderCard.tsx
'use client';

import { Button } from '@/app/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/app/components/ui/card';

interface OrderCardProps {
  order: {
    id: string;
    total: number;
    status: string;
  };
  onCancel: (orderId: string) => void;
}

export function OrderCard({ order, onCancel }: OrderCardProps) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Order #{order.id}</CardTitle>
      </CardHeader>
      <CardContent>
        <p>Total: ${order.total}</p>
        <p>Status: {order.status}</p>
        <Button onClick={() => onCancel(order.id)}>
          Cancel Order
        </Button>
      </CardContent>
    </Card>
  );
}
```

#### Rules

- ✅ Can import from Application and Infrastructure layers (via DI)
- ✅ Handles HTTP requests/responses
- ✅ Renders UI
- ❌ NO business logic (delegate to Application layer)
- ❌ NO direct Domain layer access (use Application layer)

---

## DDD Tactical Patterns

### Entities

**Characteristics:**
- Have unique identity
- Mutable state
- Equality by ID, not by attributes

**Example:**

```typescript
export class Order {
  constructor(
    private readonly id: OrderId,
    private status: OrderStatus,
    private items: OrderItem[],
    private total: Price,
    private readonly createdAt: Date
  ) {}

  addItem(item: OrderItem): void {
    // Business rule enforcement
    if (this.status === OrderStatus.COMPLETED) {
      throw new InvalidOrderException("Cannot modify completed order");
    }
    this.items.push(item);
    this.recalculateTotal();
  }

  complete(): void {
    if (this.items.length === 0) {
      throw new InvalidOrderException("Cannot complete empty order");
    }
    this.status = OrderStatus.COMPLETED;
  }

  // Equality by ID
  equals(other: Order): boolean {
    return this.id.equals(other.id);
  }
}
```

---

### Value Objects

**Characteristics:**
- No identity
- Immutable
- Equality by attributes

**Example:**

```typescript
export class Email {
  private readonly value: string;

  constructor(email: string) {
    if (!this.isValid(email)) {
      throw new InvalidEmailException("Invalid email format");
    }
    this.value = email;
  }

  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  getValue(): string {
    return this.value;
  }

  // Equality by value
  equals(other: Email): boolean {
    return this.value === other.value;
  }
}
```

---

### Aggregates

**Characteristics:**
- Cluster of entities and value objects
- Aggregate Root controls access
- Transactional consistency boundary

**Example:**

```typescript
export class Order { // Aggregate Root
  private readonly id: OrderId;
  private items: OrderItem[]; // Part of aggregate
  private total: Price;

  addItem(menuId: MenuId, quantity: number): void {
    // Enforce invariants
    const item = new OrderItem(menuId, quantity);
    this.items.push(item);
    this.recalculateTotal(); // Maintain consistency
  }

  private recalculateTotal(): void {
    let sum = 0;
    for (const item of this.items) {
      sum += item.getSubtotal();
    }
    this.total = new Price(sum);
  }

  // Only allow access to items through aggregate root
  getItems(): readonly OrderItem[] {
    return Object.freeze([...this.items]);
  }
}
```

---

### Domain Services

**Characteristics:**
- Operations that don't naturally fit in entities/value objects
- Stateless
- Coordinate multiple entities

**Example:**

```typescript
export class OrderPricingService {
  calculateDiscount(order: Order, customer: Customer): Price {
    const baseTotal = order.getTotal();

    if (customer.isPremium()) {
      return new Price(baseTotal.getValue() * 0.9); // 10% discount
    }

    if (order.getItems().length > 10) {
      return new Price(baseTotal.getValue() * 0.95); // 5% bulk discount
    }

    return baseTotal;
  }
}
```

---

### Repository Pattern

**Characteristics:**
- Interface in Domain, Implementation in Infrastructure
- Collection-like interface for aggregates
- Abstracts persistence logic

**Example:**

```typescript
// Domain layer - Interface
export interface IOrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findByCustomerId(customerId: CustomerId): Promise<Order[]>;
  save(order: Order): Promise<void>;
  delete(id: OrderId): Promise<void>;
}

// Infrastructure layer - Implementation
export class PrismaOrderRepository implements IOrderRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: OrderId): Promise<Order | null> {
    const data = await this.prisma.order.findUnique({
      where: { id: id.getValue() }
    });
    return data ? this.toDomain(data) : null;
  }

  async save(order: Order): Promise<void> {
    const data = this.toSchema(order);
    await this.prisma.order.upsert({
      where: { id: data.id },
      create: data,
      update: data
    });
  }

  private toSchema(order: Order): any {
    // Map domain entity to database schema
  }

  private toDomain(data: any): Order {
    // Map database schema to domain entity
  }
}
```

---

### Domain Events

**Characteristics:**
- Something that happened in the domain
- Past tense naming
- Immutable

**Example:**

```typescript
export class OrderPlacedEvent {
  constructor(
    public readonly orderId: OrderId,
    public readonly customerId: CustomerId,
    public readonly total: Price,
    public readonly occurredAt: Date
  ) {}
}

// Usage in entity
export class Order {
  private events: DomainEvent[] = [];

  placeOrder(): void {
    this.status = OrderStatus.PLACED;
    this.events.push(
      new OrderPlacedEvent(
        this.id,
        this.customerId,
        this.total,
        new Date()
      )
    );
  }

  getDomainEvents(): DomainEvent[] {
    return this.events;
  }

  clearDomainEvents(): void {
    this.events = [];
  }
}
```

---

## Dependency Rule

**CRITICAL: Dependencies flow inward only**

```
Presentation → Application → Domain
Infrastructure → Application → Domain
                   ↑
            (interfaces only)
```

### Layer Dependency Matrix

| Layer | Domain | Application | Infrastructure | Presentation |
|-------|--------|-------------|----------------|--------------|
| **Domain** | ✅ | ❌ | ❌ | ❌ |
| **Application** | ✅ | ✅ | ❌ | ❌ |
| **Infrastructure** | ✅ | ✅ | ✅ | ❌ |
| **Presentation** | ❌ | ✅ | ✅ | ✅ |

**Note:** Presentation should NOT directly import Domain. Use Application layer as intermediary.

---

## Bounded Contexts

**Purpose:** Organize code by business domains with clear boundaries.

### Context Organization

```
src/domain/
├── cafeteria/          # Cafeteria Context
│   ├── entities/
│   ├── value-objects/
│   ├── services/
│   └── repositories/
├── inventory/          # Inventory Context
│   ├── entities/
│   ├── value-objects/
│   ├── services/
│   └── repositories/
├── billing/            # Billing Context
│   ├── entities/
│   ├── value-objects/
│   ├── services/
│   └── repositories/
└── shared/             # Shared Kernel (cross-context)
    ├── value-objects/
    └── interfaces/
```

### Context Integration

```typescript
// Anti-Corruption Layer (ACL)
// Translates between contexts

export class InventoryAdapter {
  constructor(private inventoryService: InventoryService) {}

  async checkAvailability(menuId: MenuId): Promise<boolean> {
    // Translate cafeteria domain concept to inventory domain
    const inventoryItemId = this.toInventoryItemId(menuId);
    return await this.inventoryService.isAvailable(inventoryItemId);
  }

  private toInventoryItemId(menuId: MenuId): InventoryItemId {
    // Mapping logic
    return new InventoryItemId(menuId.getValue());
  }
}
```

---

## Ubiquitous Language

**Purpose:** Use business terminology consistently in code.

### Guidelines

1. **Collaborate with domain experts** for naming
2. **Use business terms** in class/method names
3. **Avoid technical jargon** in domain layer
4. **Document business rules** with comments

### Good Examples

```typescript
// ✅ Good: Clear business terminology
class Menu {
  markAsUnavailable(): void { }
  updatePrice(newPrice: Price): void { }
  addToSpecialOffers(): void { }
}

class Customer {
  upgradeToPremium(): void { }
  placeOrder(order: Order): void { }
}
```

### Bad Examples

```typescript
// ❌ Bad: Technical/vague terminology
class Menu {
  setFlag(value: boolean): void { }  // What flag?
  update(data: any): void { }        // Too generic
  doSomething(): void { }            // Meaningless
}

class Customer {
  changeStatus(status: number): void { }  // What does status mean?
}
```

---

## Best Practices

### 1. Rich Domain Models

**❌ Anemic Domain Model (Bad):**
```typescript
export class Order {
  id: string;
  total: number;
  status: string;

  // Only getters/setters, no behavior
  getId(): string { return this.id; }
  setTotal(total: number): void { this.total = total; }
}
```

**✅ Rich Domain Model (Good):**
```typescript
export class Order {
  constructor(
    private readonly id: OrderId,
    private total: Price,
    private status: OrderStatus
  ) {}

  // Behavior with business rules
  addItem(item: OrderItem): void {
    if (this.status !== OrderStatus.DRAFT) {
      throw new Error("Cannot modify non-draft order");
    }
    this.items.push(item);
    this.recalculateTotal();
  }
}
```

### 2. Validate Early

```typescript
export class Email {
  constructor(email: string) {
    // Validate in constructor
    if (!this.isValid(email)) {
      throw new InvalidEmailException();
    }
    this.value = email;
  }

  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}
```

### 3. Encapsulation

```typescript
export class Order {
  private items: OrderItem[] = [];

  // Return readonly copy
  getItems(): readonly OrderItem[] {
    return Object.freeze([...this.items]);
  }

  // Modify through methods with business rules
  addItem(item: OrderItem): void {
    this.validateItem(item);
    this.items.push(item);
  }
}
```

### 4. Dependency Injection

```typescript
// Application layer
export class CreateOrderUseCase {
  constructor(
    private orderRepository: IOrderRepository,  // Interface
    private menuRepository: IMenuRepository     // Interface
  ) {}

  async execute(dto: CreateOrderDto): Promise<OrderDto> {
    // Use injected dependencies
    const menu = await this.menuRepository.findById(dto.menuId);
    // ...
  }
}

// Presentation layer - Inject implementations
const useCase = new CreateOrderUseCase(
  new PrismaOrderRepository(prisma),  // Infrastructure implementation
  new PrismaMenuRepository(prisma)    // Infrastructure implementation
);
```

---

## Summary

### Key Takeaways

1. **Four Layers**: Domain → Application → Infrastructure + Presentation
2. **Dependency Rule**: Dependencies flow inward only
3. **Rich Domain Models**: Entities with behavior, not just data
4. **Value Objects**: Immutable, validated in constructor
5. **Aggregates**: Consistency boundaries
6. **Repositories**: Interface in Domain, implementation in Infrastructure
7. **Ubiquitous Language**: Use business terminology
8. **Bounded Contexts**: Clear domain boundaries

### Quick Reference

- **Business logic** → Domain layer
- **Orchestration** → Application layer
- **I/O operations** → Infrastructure layer
- **UI/API handling** → Presentation layer

---

**Back to:** [CLAUDE.md](../../CLAUDE.md)
