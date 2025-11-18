# Component Patterns

> **Version:** 2.0.0
> **Last Updated:** 2025-11-17

---

## Table of Contents

1. [Overview](#overview)
2. [Server Component Pattern](#server-component-pattern)
3. [Client Component Pattern](#client-component-pattern)
4. [Layout Pattern](#layout-pattern)
5. [shadcn/ui Component Pattern](#shadcnui-component-pattern)
6. [API Route Pattern](#api-route-pattern)
7. [Form Pattern](#form-pattern)
8. [Data Fetching Pattern](#data-fetching-pattern)

---

## Overview

This guide provides Next.js 16 component patterns for the Cafeteria Management System.

**📖 For DDD architecture and domain model details, see:**

- [`specs/01-architecture.md`](../../specs/01-architecture.md) - Architecture overview
- [`specs/02-domain-model.md`](../../specs/02-domain-model.md) - Domain entities and value objects

### Key Principles

- **Server Components by Default** - Use Server Components unless client-side interactivity is needed
- **DDD Compliance** - Components should use Application layer, not Domain directly
- **Type Safety** - All components must have TypeScript interfaces for props
- **Composition** - Build complex UIs from simple, reusable components

---

## Server Component Pattern

### When to Use

- Default for all components
- Data fetching at component level
- No client-side interactivity needed
- SEO-important content

### Basic Pattern

```typescript
// app/(dashboard)/orders/page.tsx
import type { Metadata } from 'next';
import { GetAllOrdersUseCase } from '@/src/application/cafeteria/use-cases/orders/GetAllOrdersUseCase';
import { PrismaOrderRepository } from '@/src/infrastructure/persistence/prisma/repositories/PrismaOrderRepository';
import { prisma } from '@/src/infrastructure/persistence/prisma/client';
import { OrderList } from '@/app/components/features/orders/OrderList';

export const metadata: Metadata = {
  title: 'Orders | Dashboard',
  description: 'View and manage all orders'
};

export default async function OrdersPage() {
  // Data fetching in Server Component
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

### With Dynamic Metadata

```typescript
// app/(dashboard)/orders/[id]/page.tsx
import type { Metadata } from 'next';
import { GetOrderByIdUseCase } from '@/src/application/cafeteria/use-cases/orders/GetOrderByIdUseCase';

interface OrderPageProps {
  params: { id: string };
}

export async function generateMetadata({ params }: OrderPageProps): Promise<Metadata> {
  const orderRepository = new PrismaOrderRepository(prisma);
  const useCase = new GetOrderByIdUseCase(orderRepository);
  const order = await useCase.execute(params.id);

  return {
    title: `Order #${order.id} | Dashboard`,
    description: `Order details for order #${order.id}`
  };
}

export default async function OrderPage({ params }: OrderPageProps) {
  const orderRepository = new PrismaOrderRepository(prisma);
  const useCase = new GetOrderByIdUseCase(orderRepository);
  const order = await useCase.execute(params.id);

  return (
    <div>
      <h1>Order #{order.id}</h1>
      <OrderDetails order={order} />
    </div>
  );
}
```

### With Error Handling

```typescript
// app/(dashboard)/orders/page.tsx
import { notFound } from 'next/navigation';

export default async function OrdersPage() {
  try {
    const orders = await fetchOrders();

    if (!orders || orders.length === 0) {
      return (
        <div className="text-center py-12">
          <p className="text-gray-500">No orders found</p>
        </div>
      );
    }

    return <OrderList orders={orders} />;

  } catch (error) {
    if (error instanceof NotFoundException) {
      notFound();  // Shows 404 page
    }
    throw error;  // Shows error page
  }
}
```

---

## Client Component Pattern

### When to Use

- Component uses React hooks (useState, useEffect, etc.)
- Event handlers (onClick, onChange, etc.)
- Browser APIs (localStorage, navigator, etc.)
- Third-party libraries requiring client-side rendering

### Basic Pattern

```typescript
// app/components/features/orders/OrderFilters.tsx
'use client';

import { useState } from 'react';
import { Button } from '@/app/components/ui/button';
import { Input } from '@/app/components/ui/input';

interface OrderFiltersProps {
  onFilterChange: (filters: OrderFilterCriteria) => void;
}

export function OrderFilters({ onFilterChange }: Readonly<OrderFiltersProps>) {
  const [status, setStatus] = useState<string>('all');
  const [searchTerm, setSearchTerm] = useState<string>('');

  const handleApplyFilters = () => {
    onFilterChange({
      status: status === 'all' ? undefined : status,
      searchTerm: searchTerm || undefined
    });
  };

  return (
    <div className="flex gap-4">
      <Input
        type="text"
        placeholder="Search orders..."
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
      />

      <select
        value={status}
        onChange={(e) => setStatus(e.target.value)}
        className="border rounded px-3 py-2"
      >
        <option value="all">All Status</option>
        <option value="pending">Pending</option>
        <option value="completed">Completed</option>
        <option value="cancelled">Cancelled</option>
      </select>

      <Button onClick={handleApplyFilters}>
        Apply Filters
      </Button>
    </div>
  );
}
```

### With Form Submission

```typescript
// app/components/features/orders/CreateOrderForm.tsx
'use client';

import { useState } from 'react';
import { Button } from '@/app/components/ui/button';
import { Input } from '@/app/components/ui/input';
import { useRouter } from 'next/navigation';

export function CreateOrderForm() {
  const router = useRouter();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setIsLoading(true);
    setError(null);

    const formData = new FormData(e.currentTarget);
    const data = {
      menuId: formData.get('menuId') as string,
      quantity: parseInt(formData.get('quantity') as string),
      customerId: formData.get('customerId') as string
    };

    try {
      const response = await fetch('/api/orders', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data)
      });

      if (!response.ok) {
        throw new Error('Failed to create order');
      }

      const result = await response.json();
      router.push(`/orders/${result.id}`);
      router.refresh();  // Refresh Server Components

    } catch (err) {
      setError(err instanceof Error ? err.message : 'An error occurred');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      {error && (
        <div className="bg-red-50 text-red-600 p-4 rounded">
          {error}
        </div>
      )}

      <Input
        name="menuId"
        placeholder="Menu ID"
        required
        disabled={isLoading}
      />

      <Input
        name="quantity"
        type="number"
        placeholder="Quantity"
        min="1"
        required
        disabled={isLoading}
      />

      <Input
        name="customerId"
        placeholder="Customer ID"
        required
        disabled={isLoading}
      />

      <Button type="submit" disabled={isLoading}>
        {isLoading ? 'Creating...' : 'Create Order'}
      </Button>
    </form>
  );
}
```

### Combining Server and Client Components

```typescript
// app/(dashboard)/orders/page.tsx (Server Component)
import { OrderFilters } from '@/app/components/features/orders/OrderFilters';

export default async function OrdersPage() {
  const orders = await fetchOrders();

  return (
    <div>
      <h1>Orders</h1>

      {/* Client Component for interactivity */}
      <OrderFilters onFilterChange={(filters) => {
        // Handle filter change
      }} />

      {/* Server Component for data display */}
      <OrderList orders={orders} />
    </div>
  );
}
```

---

## Layout Pattern

### Root Layout

```typescript
// app/layout.tsx
import type { Metadata } from 'next';
import { GeistSans, GeistMono } from 'geist/font';
import './globals.css';

export const metadata: Metadata = {
  title: {
    template: '%s | Cafeteria Management',
    default: 'Cafeteria Management System'
  },
  description: 'Modern cafeteria management solution'
};

export default function RootLayout({
  children
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en" className={`${GeistSans.variable} ${GeistMono.variable}`}>
      <body className="antialiased">
        {children}
      </body>
    </html>
  );
}
```

### Nested Layout

```typescript
// app/(dashboard)/layout.tsx
import { Sidebar } from '@/app/components/layouts/Sidebar';
import { Header } from '@/app/components/layouts/Header';

export default function DashboardLayout({
  children
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <div className="flex min-h-screen">
      <Sidebar />

      <div className="flex-1 flex flex-col">
        <Header />

        <main className="flex-1 p-8 bg-gray-50 dark:bg-gray-900">
          {children}
        </main>
      </div>
    </div>
  );
}
```

### Route Groups

```
app/
├── (public)/           # Public routes (no layout)
│   ├── page.tsx
│   └── menu/
│       └── page.tsx
├── (dashboard)/        # Dashboard routes (with sidebar)
│   ├── layout.tsx
│   └── orders/
│       └── page.tsx
└── (auth)/             # Auth routes (centered layout)
    ├── layout.tsx
    ├── login/
    │   └── page.tsx
    └── register/
        └── page.tsx
```

---

## shadcn/ui Component Pattern

### Adding Components

```bash
# Add a single component
npx shadcn@latest add button

# Add multiple components
npx shadcn@latest add button card dialog input
```

### Using shadcn/ui Components

```typescript
// app/components/features/orders/OrderCard.tsx
import { Button } from '@/app/components/ui/button';
import {
  Card,
  CardContent,
  CardDescription,
  CardFooter,
  CardHeader,
  CardTitle
} from '@/app/components/ui/card';

interface OrderCardProps {
  order: {
    id: string;
    total: number;
    status: string;
    createdAt: Date;
  };
  onViewDetails: (orderId: string) => void;
}

export function OrderCard({ order, onViewDetails }: Readonly<OrderCardProps>) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Order #{order.id}</CardTitle>
        <CardDescription>
          Placed on {order.createdAt.toLocaleDateString()}
        </CardDescription>
      </CardHeader>

      <CardContent>
        <div className="space-y-2">
          <p className="text-2xl font-bold">${order.total.toFixed(2)}</p>
          <p className="text-sm text-gray-500">Status: {order.status}</p>
        </div>
      </CardContent>

      <CardFooter>
        <Button onClick={() => onViewDetails(order.id)} variant="default">
          View Details
        </Button>
      </CardFooter>
    </Card>
  );
}
```

### Customizing shadcn/ui Components

```typescript
// app/components/ui/button.tsx (generated by shadcn)
// You can modify this file to add custom variants

const buttonVariants = cva(
  "inline-flex items-center justify-center...",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground...",
        destructive: "bg-destructive text-destructive-foreground...",
        outline: "border border-input...",
        // Add custom variant
        custom: "bg-purple-500 text-white hover:bg-purple-600"
      }
    }
  }
);

// Usage
<Button variant="custom">Custom Button</Button>
```

---

## API Route Pattern

### Basic GET Route

```typescript
// app/api/orders/route.ts
import { NextRequest, NextResponse } from "next/server";
import { GetAllOrdersUseCase } from "@/src/application/cafeteria/use-cases/orders/GetAllOrdersUseCase";
import { PrismaOrderRepository } from "@/src/infrastructure/persistence/prisma/repositories/PrismaOrderRepository";
import { prisma } from "@/src/infrastructure/persistence/prisma/client";

export async function GET(request: NextRequest) {
  try {
    // Dependency Injection
    const orderRepository = new PrismaOrderRepository(prisma);
    const useCase = new GetAllOrdersUseCase(orderRepository);

    // Execute use case
    const orders = await useCase.execute();

    // Return response
    return NextResponse.json(orders, { status: 200 });
  } catch (error) {
    console.error("Error fetching orders:", error);
    return NextResponse.json(
      { error: "Failed to fetch orders" },
      { status: 500 }
    );
  }
}
```

### POST Route with Validation

```typescript
// app/api/orders/route.ts
import { NextRequest, NextResponse } from "next/server";
import { CreateOrderUseCase } from "@/src/application/cafeteria/use-cases/orders/CreateOrderUseCase";
import { CreateOrderDto } from "@/src/application/cafeteria/dtos/orders/CreateOrderDto";
import { ValidationException } from "@/src/application/shared/exceptions/ValidationException";

export async function POST(request: NextRequest) {
  try {
    // 1. Parse request body
    const body = await request.json();

    // 2. Create DTO
    const dto = new CreateOrderDto(body.menuId, body.quantity, body.customerId);

    // 3. Inject dependencies
    const orderRepository = new PrismaOrderRepository(prisma);
    const menuRepository = new PrismaMenuRepository(prisma);
    const useCase = new CreateOrderUseCase(orderRepository, menuRepository);

    // 4. Execute use case
    const result = await useCase.execute(dto);

    // 5. Return response
    return NextResponse.json(result, { status: 201 });
  } catch (error) {
    if (error instanceof ValidationException) {
      return NextResponse.json({ error: error.message }, { status: 400 });
    }

    if (error instanceof NotFoundException) {
      return NextResponse.json({ error: error.message }, { status: 404 });
    }

    console.error("Error creating order:", error);
    return NextResponse.json(
      { error: "Internal server error" },
      { status: 500 }
    );
  }
}
```

### Dynamic Route

```typescript
// app/api/orders/[id]/route.ts
import { NextRequest, NextResponse } from "next/server";

interface RouteContext {
  params: { id: string };
}

export async function GET(request: NextRequest, { params }: RouteContext) {
  try {
    const orderRepository = new PrismaOrderRepository(prisma);
    const useCase = new GetOrderByIdUseCase(orderRepository);
    const order = await useCase.execute(params.id);

    if (!order) {
      return NextResponse.json({ error: "Order not found" }, { status: 404 });
    }

    return NextResponse.json(order);
  } catch (error) {
    return NextResponse.json(
      { error: "Internal server error" },
      { status: 500 }
    );
  }
}

export async function DELETE(request: NextRequest, { params }: RouteContext) {
  try {
    const orderRepository = new PrismaOrderRepository(prisma);
    const useCase = new DeleteOrderUseCase(orderRepository);
    await useCase.execute(params.id);

    return NextResponse.json({ success: true }, { status: 200 });
  } catch (error) {
    return NextResponse.json(
      { error: "Failed to delete order" },
      { status: 500 }
    );
  }
}
```

---

## Form Pattern

### Server Action Pattern

```typescript
// app/actions/createOrder.ts
"use server";

import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { CreateOrderUseCase } from "@/src/application/cafeteria/use-cases/orders/CreateOrderUseCase";

export async function createOrder(formData: FormData) {
  const menuId = formData.get("menuId") as string;
  const quantity = parseInt(formData.get("quantity") as string);
  const customerId = formData.get("customerId") as string;

  try {
    const orderRepository = new PrismaOrderRepository(prisma);
    const menuRepository = new PrismaMenuRepository(prisma);
    const useCase = new CreateOrderUseCase(orderRepository, menuRepository);

    const dto = new CreateOrderDto(menuId, quantity, customerId);
    const result = await useCase.execute(dto);

    revalidatePath("/orders"); // Revalidate cache
    redirect(`/orders/${result.id}`); // Redirect to order page
  } catch (error) {
    return {
      error: error instanceof Error ? error.message : "Failed to create order",
    };
  }
}
```

```typescript
// app/components/features/orders/CreateOrderForm.tsx
import { createOrder } from '@/app/actions/createOrder';
import { Button } from '@/app/components/ui/button';
import { Input } from '@/app/components/ui/input';

export function CreateOrderForm() {
  return (
    <form action={createOrder} className="space-y-4">
      <Input name="menuId" placeholder="Menu ID" required />
      <Input name="quantity" type="number" placeholder="Quantity" min="1" required />
      <Input name="customerId" placeholder="Customer ID" required />
      <Button type="submit">Create Order</Button>
    </form>
  );
}
```

---

## Data Fetching Pattern

### Server Component Data Fetching

```typescript
// app/(dashboard)/orders/page.tsx
export default async function OrdersPage() {
  // Fetch data directly in Server Component
  const orderRepository = new PrismaOrderRepository(prisma);
  const useCase = new GetAllOrdersUseCase(orderRepository);
  const orders = await useCase.execute();

  return <OrderList orders={orders} />;
}
```

### Parallel Data Fetching

```typescript
export default async function OrderDetailsPage({ params }: { params: { id: string } }) {
  // Fetch multiple data sources in parallel
  const [order, customer, menu] = await Promise.all([
    fetchOrder(params.id),
    fetchCustomer(params.customerId),
    fetchMenu(params.menuId)
  ]);

  return (
    <div>
      <OrderDetails order={order} />
      <CustomerInfo customer={customer} />
      <MenuInfo menu={menu} />
    </div>
  );
}
```

### Streaming with Suspense

```typescript
import { Suspense } from 'react';

export default function OrdersPage() {
  return (
    <div>
      <h1>Orders</h1>

      <Suspense fallback={<OrderListSkeleton />}>
        <OrderList />
      </Suspense>
    </div>
  );
}

async function OrderList() {
  const orders = await fetchOrders();  // Async data fetching
  return <div>{/* Render orders */}</div>;
}
```

---

## Summary

### Quick Reference

| Pattern              | Use When                       | Example                      |
| -------------------- | ------------------------------ | ---------------------------- |
| **Server Component** | Default, data fetching, SEO    | `app/orders/page.tsx`        |
| **Client Component** | Hooks, events, browser APIs    | `OrderFilters.tsx`           |
| **Layout**           | Shared UI across routes        | `app/(dashboard)/layout.tsx` |
| **shadcn/ui**        | Pre-built UI components        | `<Button>`, `<Card>`         |
| **API Route**        | REST API endpoints             | `app/api/orders/route.ts`    |
| **Server Action**    | Form submissions               | `createOrder()`              |
| **Data Fetching**    | Load data in Server Components | `await fetchOrders()`        |

### Best Practices

- ✅ Use Server Components by default
- ✅ Add "use client" only when necessary
- ✅ Fetch data in Server Components
- ✅ Use shadcn/ui for consistent UI
- ✅ Follow DDD: use Application layer, not Domain directly
- ✅ Handle errors gracefully
- ✅ Add loading states

---

**Back to:** [CLAUDE.md](../../CLAUDE.md)
