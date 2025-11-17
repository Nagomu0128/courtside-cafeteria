# DDD Layer Rules and File Organization

> **Version:** 1.0.0
> **Last Updated:** 2025-11-17

---

## Table of Contents

1. [Overview](#overview)
2. [Domain Layer Organization](#domain-layer-organization)
3. [Application Layer Organization](#application-layer-organization)
4. [Infrastructure Layer Organization](#infrastructure-layer-organization)
5. [Presentation Layer Organization](#presentation-layer-organization)
6. [Import Path Guidelines](#import-path-guidelines)
7. [Naming Conventions](#naming-conventions)

---

## Overview

**All file organization follows DDD layered architecture principles.**

This document provides detailed rules for organizing files within each DDD layer, including naming conventions, directory structure, and import guidelines.

---

## Domain Layer Organization

### Location

`/src/domain/<bounded-context>/`

### Directory Structure

```
domain/
├── cafeteria/              # Bounded Context
│   ├── entities/          # Domain Entities
│   │   ├── Menu.ts
│   │   ├── Order.ts
│   │   ├── Customer.ts
│   │   └── index.ts       # Re-export all entities
│   ├── value-objects/     # Value Objects
│   │   ├── Price.ts
│   │   ├── MenuId.ts
│   │   ├── OrderId.ts
│   │   ├── Email.ts
│   │   └── index.ts
│   ├── services/          # Domain Services
│   │   ├── OrderPricingService.ts
│   │   ├── MenuAvailabilityService.ts
│   │   └── index.ts
│   ├── repositories/      # Repository Interfaces ONLY
│   │   ├── IMenuRepository.ts
│   │   ├── IOrderRepository.ts
│   │   ├── ICustomerRepository.ts
│   │   └── index.ts
│   ├── events/            # Domain Events
│   │   ├── OrderPlacedEvent.ts
│   │   ├── MenuUpdatedEvent.ts
│   │   └── index.ts
│   ├── exceptions/        # Domain Exceptions
│   │   ├── InvalidOrderException.ts
│   │   ├── InvalidPriceException.ts
│   │   └── index.ts
│   └── types/             # Domain-specific TypeScript types
│       ├── OrderStatus.ts
│       └── index.ts
├── inventory/             # Another Bounded Context
│   └── ...
└── shared/                # Shared Kernel (cross-context)
    ├── value-objects/
    │   ├── Money.ts
    │   └── DateRange.ts
    └── interfaces/
        └── IRepository.ts
```

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| **Entities** | PascalCase, business nouns | `Order.ts`, `Menu.ts`, `Customer.ts` |
| **Value Objects** | PascalCase, descriptive nouns | `Price.ts`, `Email.ts`, `Address.ts` |
| **Repository Interfaces** | `I<Entity>Repository` | `IOrderRepository.ts`, `IMenuRepository.ts` |
| **Domain Services** | `<Concept>Service` | `OrderPricingService.ts`, `InventoryService.ts` |
| **Domain Events** | Past tense | `OrderPlacedEvent.ts`, `MenuUpdatedEvent.ts` |
| **Exceptions** | `<Concept>Exception` | `InvalidOrderException.ts` |

### File Organization Rules

#### Entities

```typescript
// src/domain/cafeteria/entities/Order.ts
import { OrderId } from '../value-objects/OrderId';
import { Price } from '../value-objects/Price';
import { OrderStatus } from '../types/OrderStatus';

export class Order {
  constructor(
    private readonly id: OrderId,
    private status: OrderStatus,
    private items: OrderItem[]
  ) {}

  // Business logic methods
  addItem(item: OrderItem): void {
    if (this.isCompleted()) {
      throw new InvalidOrderException("Cannot modify completed order");
    }
    this.items.push(item);
  }
}
```

#### Value Objects

```typescript
// src/domain/cafeteria/value-objects/Price.ts
export class Price {
  private readonly amount: number;

  constructor(amount: number) {
    if (amount < 0) {
      throw new InvalidPriceException("Price cannot be negative");
    }
    this.amount = amount;
  }

  getValue(): number {
    return this.amount;
  }

  equals(other: Price): boolean {
    return this.amount === other.amount;
  }
}
```

#### Repository Interfaces

```typescript
// src/domain/cafeteria/repositories/IOrderRepository.ts
import { Order } from '../entities/Order';
import { OrderId } from '../value-objects/OrderId';

export interface IOrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findAll(): Promise<Order[]>;
  save(order: Order): Promise<void>;
  delete(id: OrderId): Promise<void>;
}
```

#### Index Files

```typescript
// src/domain/cafeteria/entities/index.ts
export { Order } from './Order';
export { Menu } from './Menu';
export { Customer } from './Customer';
```

### Layer Rules

- ✅ **Pure business logic** only
- ✅ **Framework-agnostic** TypeScript (no Next.js, React, etc.)
- ✅ **Use Ubiquitous Language** from business domain
- ✅ **Validate in constructors** (fail fast)
- ❌ **NO imports from other layers** (Application, Infrastructure, Presentation)
- ❌ **NO I/O operations** (database, HTTP, file system)
- ❌ **NO framework dependencies** (React, Next.js, Prisma, etc.)

---

## Application Layer Organization

### Location

`/src/application/<bounded-context>/`

### Directory Structure

```
application/
├── cafeteria/
│   ├── use-cases/             # Use Case implementations
│   │   ├── orders/
│   │   │   ├── CreateOrderUseCase.ts
│   │   │   ├── UpdateOrderUseCase.ts
│   │   │   ├── CancelOrderUseCase.ts
│   │   │   └── GetOrderByIdUseCase.ts
│   │   ├── menu/
│   │   │   ├── UpdateMenuUseCase.ts
│   │   │   ├── GetMenuListUseCase.ts
│   │   │   └── ArchiveMenuUseCase.ts
│   │   └── index.ts
│   ├── dtos/                  # Data Transfer Objects
│   │   ├── orders/
│   │   │   ├── CreateOrderDto.ts
│   │   │   ├── UpdateOrderDto.ts
│   │   │   └── OrderDto.ts
│   │   ├── menu/
│   │   │   ├── MenuDto.ts
│   │   │   └── UpdateMenuDto.ts
│   │   └── index.ts
│   ├── services/              # Application Services
│   │   ├── OrderApplicationService.ts
│   │   ├── MenuApplicationService.ts
│   │   └── index.ts
│   ├── ports/                 # Hexagonal Architecture Ports
│   │   ├── input/
│   │   │   └── IOrderUseCase.ts
│   │   └── output/
│   │       └── IEmailService.ts
│   └── validators/            # Application-level validation
│       ├── CreateOrderValidator.ts
│       └── index.ts
└── shared/
    ├── interfaces/
    │   ├── IUseCase.ts
    │   └── IMapper.ts
    └── exceptions/
        ├── ValidationException.ts
        └── NotFoundException.ts
```

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| **Use Cases** | `<Verb><Entity>UseCase` | `CreateOrderUseCase.ts`, `GetMenuListUseCase.ts` |
| **DTOs** | `<Purpose>Dto` | `CreateOrderDto.ts`, `OrderDto.ts`, `UpdateMenuDto.ts` |
| **Application Services** | `<Entity>ApplicationService` | `OrderApplicationService.ts` |
| **Validators** | `<Purpose>Validator` | `CreateOrderValidator.ts` |

### File Organization Rules

#### Use Cases

```typescript
// src/application/cafeteria/use-cases/orders/CreateOrderUseCase.ts
import { IOrderRepository } from '@/src/domain/cafeteria/repositories/IOrderRepository';
import { IMenuRepository } from '@/src/domain/cafeteria/repositories/IMenuRepository';
import { Order } from '@/src/domain/cafeteria/entities/Order';
import { CreateOrderDto } from '../../dtos/orders/CreateOrderDto';
import { OrderDto } from '../../dtos/orders/OrderDto';

export class CreateOrderUseCase {
  constructor(
    private orderRepository: IOrderRepository,
    private menuRepository: IMenuRepository
  ) {}

  async execute(dto: CreateOrderDto): Promise<OrderDto> {
    // 1. Validate
    if (!dto.menuId || !dto.quantity) {
      throw new ValidationException("Invalid input");
    }

    // 2. Load domain entities
    const menu = await this.menuRepository.findById(dto.menuId);
    if (!menu) {
      throw new NotFoundException("Menu not found");
    }

    // 3. Execute domain logic
    const order = Order.create(menu, dto.quantity);

    // 4. Persist
    await this.orderRepository.save(order);

    // 5. Return DTO
    return OrderDto.fromEntity(order);
  }
}
```

#### DTOs

```typescript
// src/application/cafeteria/dtos/orders/CreateOrderDto.ts
export class CreateOrderDto {
  constructor(
    public readonly menuId: string,
    public readonly quantity: number,
    public readonly customerId: string
  ) {}
}

// src/application/cafeteria/dtos/orders/OrderDto.ts
import { Order } from '@/src/domain/cafeteria/entities/Order';

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

### Layer Rules

- ✅ **Orchestrate domain objects** (no business logic)
- ✅ **Import from Domain layer only**
- ✅ **Define DTOs** for data transfer
- ✅ **Depend on interfaces** (not implementations)
- ❌ **NO business logic** (delegate to Domain)
- ❌ **NO direct database access** (use repositories)
- ❌ **NO imports from Infrastructure or Presentation**

---

## Infrastructure Layer Organization

### Location

`/src/infrastructure/`

### Directory Structure

```
infrastructure/
├── persistence/
│   ├── prisma/
│   │   ├── schema.prisma
│   │   ├── repositories/
│   │   │   ├── PrismaOrderRepository.ts
│   │   │   ├── PrismaMenuRepository.ts
│   │   │   └── PrismaCustomerRepository.ts
│   │   ├── mappers/
│   │   │   ├── OrderMapper.ts
│   │   │   └── MenuMapper.ts
│   │   ├── migrations/
│   │   └── seed.ts
│   └── typeorm/              # Alternative ORM
├── external-services/
│   ├── payment/
│   │   ├── StripePaymentService.ts
│   │   └── PayPalPaymentService.ts
│   ├── notification/
│   │   ├── EmailNotificationService.ts
│   │   └── SmsNotificationService.ts
│   └── index.ts
├── messaging/
│   ├── EventBus.ts
│   ├── RabbitMQAdapter.ts
│   └── index.ts
├── config/
│   ├── database.config.ts
│   ├── redis.config.ts
│   └── index.ts
└── shared/
    └── BaseRepository.ts
```

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| **Repository Implementations** | `<Technology><Entity>Repository` | `PrismaOrderRepository.ts`, `TypeOrmMenuRepository.ts` |
| **External Services** | `<Provider><Service>Service` | `StripePaymentService.ts`, `SendGridEmailService.ts` |
| **Mappers** | `<Entity>Mapper` | `OrderMapper.ts`, `MenuMapper.ts` |
| **Adapters** | `<Technology>Adapter` | `RabbitMQAdapter.ts`, `RedisAdapter.ts` |

### File Organization Rules

#### Repository Implementations

```typescript
// src/infrastructure/persistence/prisma/repositories/PrismaOrderRepository.ts
import { PrismaClient } from '@prisma/client';
import { IOrderRepository } from '@/src/domain/cafeteria/repositories/IOrderRepository';
import { Order } from '@/src/domain/cafeteria/entities/Order';
import { OrderId } from '@/src/domain/cafeteria/value-objects/OrderId';
import { OrderMapper } from '../mappers/OrderMapper';

export class PrismaOrderRepository implements IOrderRepository {
  constructor(
    private prisma: PrismaClient,
    private mapper: OrderMapper
  ) {}

  async findById(id: OrderId): Promise<Order | null> {
    const data = await this.prisma.order.findUnique({
      where: { id: id.getValue() },
      include: { items: true }
    });

    return data ? this.mapper.toDomain(data) : null;
  }

  async save(order: Order): Promise<void> {
    const data = this.mapper.toSchema(order);

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
}
```

#### Mappers

```typescript
// src/infrastructure/persistence/prisma/mappers/OrderMapper.ts
import { Order } from '@/src/domain/cafeteria/entities/Order';
import { OrderId } from '@/src/domain/cafeteria/value-objects/OrderId';
import { Price } from '@/src/domain/cafeteria/value-objects/Price';
import { OrderStatus } from '@/src/domain/cafeteria/types/OrderStatus';

export class OrderMapper {
  // Domain → Database Schema
  toSchema(order: Order): any {
    return {
      id: order.getId().getValue(),
      total: order.getTotal().getValue(),
      status: order.getStatus().toString(),
      customerId: order.getCustomerId().getValue(),
      createdAt: order.getCreatedAt(),
      updatedAt: new Date()
    };
  }

  // Database Schema → Domain
  toDomain(data: any): Order {
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

#### External Services

```typescript
// src/infrastructure/external-services/payment/StripePaymentService.ts
import Stripe from 'stripe';
import { Price } from '@/src/domain/cafeteria/value-objects/Price';
import { CustomerId } from '@/src/domain/cafeteria/value-objects/CustomerId';

export class StripePaymentService {
  private stripe: Stripe;

  constructor(apiKey: string) {
    this.stripe = new Stripe(apiKey, {
      apiVersion: '2023-10-16'
    });
  }

  async processPayment(
    amount: Price,
    customerId: CustomerId
  ): Promise<PaymentResult> {
    const intent = await this.stripe.paymentIntents.create({
      amount: amount.getValue() * 100, // Convert to cents
      currency: 'usd',
      customer: customerId.getValue()
    });

    return new PaymentResult(intent.id, intent.status);
  }
}
```

### Layer Rules

- ✅ **Implement Domain repository interfaces**
- ✅ **All I/O operations** (database, HTTP, file system)
- ✅ **Framework/library specific code**
- ✅ **Can import from Domain and Application layers**
- ❌ **NO business logic**
- ❌ **NO imports from Presentation layer**

---

## Presentation Layer Organization

### Location

`/app/` (Next.js App Router)

### Directory Structure

```
app/
├── (routes)/              # Route groups
│   ├── (public)/          # Public routes
│   │   ├── page.tsx
│   │   ├── menu/
│   │   │   └── page.tsx
│   │   └── about/
│   │       └── page.tsx
│   ├── (dashboard)/       # Dashboard routes (auth required)
│   │   ├── layout.tsx
│   │   ├── orders/
│   │   │   ├── page.tsx
│   │   │   ├── [id]/
│   │   │   │   └── page.tsx
│   │   │   └── components/
│   │   │       ├── OrderList.tsx
│   │   │       └── OrderFilters.tsx
│   │   ├── menu/
│   │   │   └── page.tsx
│   │   └── inventory/
│   │       └── page.tsx
│   └── (auth)/            # Auth routes
│       ├── login/
│       │   └── page.tsx
│       └── register/
│           └── page.tsx
├── api/                   # API Routes
│   ├── orders/
│   │   ├── route.ts
│   │   └── [id]/
│   │       └── route.ts
│   ├── menu/
│   │   ├── route.ts
│   │   └── [id]/
│   │       └── route.ts
│   └── health/
│       └── route.ts
├── components/
│   ├── ui/                # shadcn/ui components
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── dialog.tsx
│   │   └── input.tsx
│   ├── features/          # Feature components (by context)
│   │   ├── orders/
│   │   │   ├── OrderCard.tsx
│   │   │   ├── OrderForm.tsx
│   │   │   └── OrderList.tsx
│   │   ├── menu/
│   │   │   ├── MenuCard.tsx
│   │   │   └── MenuList.tsx
│   │   └── customers/
│   │       └── CustomerProfile.tsx
│   ├── shared/            # Shared custom components
│   │   ├── Header.tsx
│   │   ├── Footer.tsx
│   │   ├── Navigation.tsx
│   │   └── ErrorBoundary.tsx
│   └── layouts/
│       ├── DashboardLayout.tsx
│       ├── PublicLayout.tsx
│       └── AuthLayout.tsx
├── layout.tsx             # Root layout
├── page.tsx               # Home page
├── globals.css            # Global styles
└── error.tsx              # Error page
```

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| **Pages** | `page.tsx` | `app/dashboard/page.tsx` |
| **Layouts** | `layout.tsx` | `app/dashboard/layout.tsx` |
| **API Routes** | `route.ts` | `app/api/orders/route.ts` |
| **Components** | PascalCase | `OrderCard.tsx`, `MenuList.tsx` |
| **shadcn/ui** | lowercase | `button.tsx`, `card.tsx` |

### File Organization Rules

#### API Routes (Controllers)

```typescript
// app/api/orders/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { CreateOrderUseCase } from '@/src/application/cafeteria/use-cases/orders/CreateOrderUseCase';
import { CreateOrderDto } from '@/src/application/cafeteria/dtos/orders/CreateOrderDto';
import { PrismaOrderRepository } from '@/src/infrastructure/persistence/prisma/repositories/PrismaOrderRepository';
import { PrismaMenuRepository } from '@/src/infrastructure/persistence/prisma/repositories/PrismaMenuRepository';
import { prisma } from '@/src/infrastructure/persistence/prisma/client';

export async function POST(request: NextRequest) {
  try {
    // 1. Parse request body
    const body = await request.json();

    // 2. Create DTO
    const dto = new CreateOrderDto(
      body.menuId,
      body.quantity,
      body.customerId
    );

    // 3. Dependency Injection
    const orderRepository = new PrismaOrderRepository(prisma);
    const menuRepository = new PrismaMenuRepository(prisma);
    const useCase = new CreateOrderUseCase(orderRepository, menuRepository);

    // 4. Execute use case
    const result = await useCase.execute(dto);

    // 5. Return response
    return NextResponse.json(result, { status: 201 });

  } catch (error) {
    if (error instanceof ValidationException) {
      return NextResponse.json(
        { error: error.message },
        { status: 400 }
      );
    }
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}

export async function GET() {
  // Get all orders
  const useCase = new GetAllOrdersUseCase(orderRepository);
  const orders = await useCase.execute();
  return NextResponse.json(orders);
}
```

#### Server Components

```typescript
// app/(dashboard)/orders/page.tsx
import { GetAllOrdersUseCase } from '@/src/application/cafeteria/use-cases/orders/GetAllOrdersUseCase';
import { PrismaOrderRepository } from '@/src/infrastructure/persistence/prisma/repositories/PrismaOrderRepository';
import { prisma } from '@/src/infrastructure/persistence/prisma/client';
import { OrderList } from '@/app/components/features/orders/OrderList';

export default async function OrdersPage() {
  // Fetch data in Server Component
  const orderRepository = new PrismaOrderRepository(prisma);
  const useCase = new GetAllOrdersUseCase(orderRepository);
  const orders = await useCase.execute();

  return (
    <div className="container mx-auto py-8">
      <h1 className="text-3xl font-bold mb-6">Orders</h1>
      <OrderList orders={orders} />
    </div>
  );
}
```

#### Client Components

```typescript
// app/components/features/orders/OrderCard.tsx
'use client';

import { useState } from 'react';
import { Button } from '@/app/components/ui/button';
import { Card, CardContent, CardFooter, CardHeader, CardTitle } from '@/app/components/ui/card';

interface OrderCardProps {
  order: {
    id: string;
    total: number;
    status: string;
  };
}

export function OrderCard({ order }: OrderCardProps) {
  const [isLoading, setIsLoading] = useState(false);

  const handleCancel = async () => {
    setIsLoading(true);
    try {
      await fetch(`/api/orders/${order.id}`, {
        method: 'DELETE'
      });
      // Handle success
    } catch (error) {
      // Handle error
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <Card>
      <CardHeader>
        <CardTitle>Order #{order.id}</CardTitle>
      </CardHeader>
      <CardContent>
        <p>Total: ${order.total}</p>
        <p>Status: {order.status}</p>
      </CardContent>
      <CardFooter>
        <Button onClick={handleCancel} disabled={isLoading}>
          Cancel Order
        </Button>
      </CardFooter>
    </Card>
  );
}
```

### Layer Rules

- ✅ **Import from Application layer** (use cases)
- ✅ **Inject Infrastructure implementations** via DI
- ✅ **Handle HTTP/UI concerns only**
- ✅ **Use shadcn/ui components** from `components/ui/`
- ❌ **NO business logic** (delegate to Application layer)
- ❌ **NO direct Domain layer imports** (use Application as intermediary)
- ❌ **NO direct database access**

---

## Import Path Guidelines

### Path Aliases

Configure in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./*"],
      "@/src/domain/*": ["./src/domain/*"],
      "@/src/application/*": ["./src/application/*"],
      "@/src/infrastructure/*": ["./src/infrastructure/*"],
      "@/app/*": ["./app/*"]
    }
  }
}
```

### Import Examples

#### ✅ Correct Imports

```typescript
// Presentation → Application
import { CreateOrderUseCase } from "@/src/application/cafeteria/use-cases/orders/CreateOrderUseCase";

// Application → Domain
import { Order } from "@/src/domain/cafeteria/entities/Order";

// Infrastructure → Domain (interface)
import { IOrderRepository } from "@/src/domain/cafeteria/repositories/IOrderRepository";

// Infrastructure → Application
import { CreateOrderDto } from "@/src/application/cafeteria/dtos/orders/CreateOrderDto";
```

#### ❌ Wrong Imports

```typescript
// ❌ Application importing Infrastructure
import { PrismaOrderRepository } from "@/src/infrastructure/persistence/prisma/repositories/PrismaOrderRepository";

// ❌ Presentation importing Domain directly
import { Order } from "@/src/domain/cafeteria/entities/Order";

// ❌ Domain importing Application
import { CreateOrderDto } from "@/src/application/cafeteria/dtos/orders/CreateOrderDto";

// ❌ Domain importing Infrastructure
import { PrismaClient } from "@prisma/client";
```

### Import Order

Follow this order for cleaner code:

```typescript
// 1. External libraries
import { PrismaClient } from '@prisma/client';
import { NextRequest, NextResponse } from 'next/server';

// 2. Domain layer
import { Order } from '@/src/domain/cafeteria/entities/Order';
import { IOrderRepository } from '@/src/domain/cafeteria/repositories/IOrderRepository';

// 3. Application layer
import { CreateOrderUseCase } from '@/src/application/cafeteria/use-cases/orders/CreateOrderUseCase';
import { CreateOrderDto } from '@/src/application/cafeteria/dtos/orders/CreateOrderDto';

// 4. Infrastructure layer
import { PrismaOrderRepository } from '@/src/infrastructure/persistence/prisma/repositories/PrismaOrderRepository';

// 5. Presentation layer
import { Button } from '@/app/components/ui/button';

// 6. Relative imports (same directory)
import { helper } from './helper';
```

---

## Naming Conventions

### Summary Table

| Type | Layer | Convention | Example |
|------|-------|-----------|---------|
| Entity | Domain | PascalCase noun | `Order.ts` |
| Value Object | Domain | PascalCase noun | `Price.ts` |
| Repository Interface | Domain | `I<Entity>Repository` | `IOrderRepository.ts` |
| Domain Service | Domain | `<Concept>Service` | `OrderPricingService.ts` |
| Domain Event | Domain | Past tense | `OrderPlacedEvent.ts` |
| Use Case | Application | `<Verb><Entity>UseCase` | `CreateOrderUseCase.ts` |
| DTO | Application | `<Purpose>Dto` | `CreateOrderDto.ts` |
| Repository Impl | Infrastructure | `<Tech><Entity>Repository` | `PrismaOrderRepository.ts` |
| External Service | Infrastructure | `<Provider><Service>Service` | `StripePaymentService.ts` |
| Page | Presentation | `page.tsx` | `app/orders/page.tsx` |
| API Route | Presentation | `route.ts` | `app/api/orders/route.ts` |
| Component | Presentation | PascalCase | `OrderCard.tsx` |

---

## Summary

### Key Points

1. **Domain Layer**: Pure business logic, no external dependencies
2. **Application Layer**: Orchestration, depends on Domain only
3. **Infrastructure Layer**: Implements Domain interfaces, all I/O operations
4. **Presentation Layer**: UI/API handling, uses Application layer

### Quick Reference

- Business logic → `src/domain/<context>/entities/`
- Use cases → `src/application/<context>/use-cases/`
- DB repositories → `src/infrastructure/persistence/<orm>/repositories/`
- API routes → `app/api/<resource>/route.ts`
- UI components → `app/components/features/<context>/`

---

**Back to:** [CLAUDE.md](../../CLAUDE.md) | **Related:** [DDD Guide](./ddd-guide.md)
