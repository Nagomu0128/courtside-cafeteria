# Common Development Tasks

> **Version:** 2.0.0
> **Last Updated:** 2025-11-17

---

**📖 Prerequisites:**

- Read [`specs/01-architecture.md`](../../specs/01-architecture.md) for DDD architecture overview
- Read [`specs/02-domain-model.md`](../../specs/02-domain-model.md) for domain model examples

---

## Table of Contents

1. [Adding a New Feature (DDD Flow)](#adding-a-new-feature-ddd-flow)
2. [Adding a New Page](#adding-a-new-page)
3. [Adding a Component](#adding-a-component)
4. [Adding an API Route](#adding-an-api-route)
5. [Adding shadcn/ui Components](#adding-shadcnui-components)
6. [Formatting Code with Prettier](#formatting-code-with-prettier)
7. [Database Operations](#database-operations)
8. [Styling Tasks](#styling-tasks)
9. [Troubleshooting](#troubleshooting)

---

## Adding a New Feature (DDD Flow)

**Follow this DDD workflow when adding new features.**

### Step 1: Define Domain Entity/Value Object

```bash
# Create entity file
touch src/domain/cafeteria/entities/Reservation.ts
```

```typescript
// src/domain/cafeteria/entities/Reservation.ts
import { ReservationId } from "../value-objects/ReservationId";
import { ReservationStatus } from "../types/ReservationStatus";

export class Reservation {
  constructor(
    private readonly id: ReservationId,
    private status: ReservationStatus,
    private readonly customerId: CustomerId,
    private readonly tableNumber: number,
    private readonly reservedAt: Date
  ) {
    // Validate invariants
    if (tableNumber <= 0) {
      throw new InvalidReservationException("Table number must be positive");
    }
  }

  confirm(): void {
    if (this.status !== ReservationStatus.PENDING) {
      throw new InvalidReservationException(
        "Only pending reservations can be confirmed"
      );
    }
    this.status = ReservationStatus.CONFIRMED;
  }

  cancel(): void {
    if (this.status === ReservationStatus.CANCELLED) {
      throw new InvalidReservationException("Reservation already cancelled");
    }
    this.status = ReservationStatus.CANCELLED;
  }

  // Getters
  getId(): ReservationId {
    return this.id;
  }
  getStatus(): ReservationStatus {
    return this.status;
  }
}
```

### Step 2: Create Value Objects

```typescript
// src/domain/cafeteria/value-objects/ReservationId.ts
export class ReservationId {
  constructor(private readonly value: string) {
    if (!value || value.trim() === "") {
      throw new Error("ReservationId cannot be empty");
    }
  }

  getValue(): string {
    return this.value;
  }

  equals(other: ReservationId): boolean {
    return this.value === other.value;
  }
}
```

### Step 3: Create Repository Interface (Domain)

```typescript
// src/domain/cafeteria/repositories/IReservationRepository.ts
import { Reservation } from "../entities/Reservation";
import { ReservationId } from "../value-objects/ReservationId";

export interface IReservationRepository {
  findById(id: ReservationId): Promise<Reservation | null>;
  findByCustomerId(customerId: CustomerId): Promise<Reservation[]>;
  save(reservation: Reservation): Promise<void>;
  delete(id: ReservationId): Promise<void>;
}
```

### Step 4: Create Use Case (Application Layer)

```typescript
// src/application/cafeteria/use-cases/reservations/CreateReservationUseCase.ts
import { IReservationRepository } from "@/src/domain/cafeteria/repositories/IReservationRepository";
import { Reservation } from "@/src/domain/cafeteria/entities/Reservation";
import { CreateReservationDto } from "../../dtos/reservations/CreateReservationDto";
import { ReservationDto } from "../../dtos/reservations/ReservationDto";

export class CreateReservationUseCase {
  constructor(private reservationRepository: IReservationRepository) {}

  async execute(dto: CreateReservationDto): Promise<ReservationDto> {
    // 1. Validate
    if (!dto.customerId || !dto.tableNumber) {
      throw new ValidationException("Invalid input");
    }

    // 2. Create domain entity
    const reservation = Reservation.create(
      dto.customerId,
      dto.tableNumber,
      new Date(dto.reservedAt)
    );

    // 3. Persist
    await this.reservationRepository.save(reservation);

    // 4. Return DTO
    return ReservationDto.fromEntity(reservation);
  }
}
```

### Step 5: Create DTOs

```typescript
// src/application/cafeteria/dtos/reservations/CreateReservationDto.ts
export class CreateReservationDto {
  constructor(
    public readonly customerId: string,
    public readonly tableNumber: number,
    public readonly reservedAt: string
  ) {}
}

// src/application/cafeteria/dtos/reservations/ReservationDto.ts
export class ReservationDto {
  constructor(
    public readonly id: string,
    public readonly customerId: string,
    public readonly tableNumber: number,
    public readonly status: string,
    public readonly reservedAt: Date
  ) {}

  static fromEntity(reservation: Reservation): ReservationDto {
    return new ReservationDto(
      reservation.getId().getValue(),
      reservation.getCustomerId().getValue(),
      reservation.getTableNumber(),
      reservation.getStatus().toString(),
      reservation.getReservedAt()
    );
  }
}
```

### Step 6: Implement Repository (Infrastructure)

```typescript
// src/infrastructure/persistence/prisma/repositories/PrismaReservationRepository.ts
import { PrismaClient } from "@prisma/client";
import { IReservationRepository } from "@/src/domain/cafeteria/repositories/IReservationRepository";
import { Reservation } from "@/src/domain/cafeteria/entities/Reservation";
import { ReservationId } from "@/src/domain/cafeteria/value-objects/ReservationId";

export class PrismaReservationRepository implements IReservationRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: ReservationId): Promise<Reservation | null> {
    const data = await this.prisma.reservation.findUnique({
      where: { id: id.getValue() },
    });
    return data ? this.toDomain(data) : null;
  }

  async save(reservation: Reservation): Promise<void> {
    const data = this.toSchema(reservation);
    await this.prisma.reservation.upsert({
      where: { id: data.id },
      create: data,
      update: data,
    });
  }

  private toSchema(reservation: Reservation): any {
    return {
      id: reservation.getId().getValue(),
      customerId: reservation.getCustomerId().getValue(),
      tableNumber: reservation.getTableNumber(),
      status: reservation.getStatus().toString(),
      reservedAt: reservation.getReservedAt(),
    };
  }

  private toDomain(data: any): Reservation {
    return new Reservation(
      new ReservationId(data.id),
      new ReservationStatus(data.status),
      new CustomerId(data.customerId),
      data.tableNumber,
      data.reservedAt
    );
  }
}
```

### Step 7: Create API Route (Presentation)

```typescript
// app/api/reservations/route.ts
import { NextRequest, NextResponse } from "next/server";
import { CreateReservationUseCase } from "@/src/application/cafeteria/use-cases/reservations/CreateReservationUseCase";
import { PrismaReservationRepository } from "@/src/infrastructure/persistence/prisma/repositories/PrismaReservationRepository";
import { prisma } from "@/src/infrastructure/persistence/prisma/client";

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();

    const reservationRepository = new PrismaReservationRepository(prisma);
    const useCase = new CreateReservationUseCase(reservationRepository);

    const dto = new CreateReservationDto(
      body.customerId,
      body.tableNumber,
      body.reservedAt
    );

    const result = await useCase.execute(dto);

    return NextResponse.json(result, { status: 201 });
  } catch (error) {
    return NextResponse.json({ error: error.message }, { status: 400 });
  }
}
```

### Step 8: Create UI Component

```typescript
// app/components/features/reservations/CreateReservationForm.tsx
'use client';

import { useState } from 'react';
import { Button } from '@/app/components/ui/button';
import { Input } from '@/app/components/ui/input';

export function CreateReservationForm() {
  const [isLoading, setIsLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setIsLoading(true);

    const formData = new FormData(e.currentTarget);
    const data = {
      customerId: formData.get('customerId') as string,
      tableNumber: parseInt(formData.get('tableNumber') as string),
      reservedAt: formData.get('reservedAt') as string
    };

    try {
      const response = await fetch('/api/reservations', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data)
      });

      if (!response.ok) throw new Error('Failed to create reservation');

      // Handle success
      alert('Reservation created successfully!');
      e.currentTarget.reset();

    } catch (error) {
      alert('Error: ' + error.message);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <Input name="customerId" placeholder="Customer ID" required />
      <Input name="tableNumber" type="number" placeholder="Table Number" required />
      <Input name="reservedAt" type="datetime-local" required />
      <Button type="submit" disabled={isLoading}>
        {isLoading ? 'Creating...' : 'Create Reservation'}
      </Button>
    </form>
  );
}
```

---

## Adding a New Page

### Server Component Page

```bash
# Create page directory
mkdir -p app/(dashboard)/reservations

# Create page file
touch app/(dashboard)/reservations/page.tsx
```

```typescript
// app/(dashboard)/reservations/page.tsx
import type { Metadata } from 'next';
import { GetAllReservationsUseCase } from '@/src/application/cafeteria/use-cases/reservations/GetAllReservationsUseCase';
import { PrismaReservationRepository } from '@/src/infrastructure/persistence/prisma/repositories/PrismaReservationRepository';
import { prisma } from '@/src/infrastructure/persistence/prisma/client';

export const metadata: Metadata = {
  title: 'Reservations | Dashboard',
  description: 'View and manage reservations'
};

export default async function ReservationsPage() {
  const reservationRepository = new PrismaReservationRepository(prisma);
  const useCase = new GetAllReservationsUseCase(reservationRepository);
  const reservations = await useCase.execute();

  return (
    <div className="container mx-auto py-8">
      <h1 className="text-3xl font-bold mb-6">Reservations</h1>
      <ReservationList reservations={reservations} />
    </div>
  );
}
```

### Dynamic Page

```bash
# Create dynamic route
mkdir -p app/(dashboard)/reservations/[id]
touch app/(dashboard)/reservations/[id]/page.tsx
```

```typescript
// app/(dashboard)/reservations/[id]/page.tsx
interface ReservationPageProps {
  params: { id: string };
}

export async function generateMetadata({ params }: ReservationPageProps): Promise<Metadata> {
  return {
    title: `Reservation #${params.id} | Dashboard`
  };
}

export default async function ReservationPage({ params }: ReservationPageProps) {
  const reservation = await fetchReservation(params.id);

  return (
    <div>
      <h1>Reservation #{reservation.id}</h1>
      <ReservationDetails reservation={reservation} />
    </div>
  );
}
```

---

## Adding a Component

### Client Component

```bash
# Create component file
touch app/components/features/reservations/ReservationCard.tsx
```

```typescript
// app/components/features/reservations/ReservationCard.tsx
'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/app/components/ui/card';
import { Button } from '@/app/components/ui/button';

interface ReservationCardProps {
  reservation: {
    id: string;
    tableNumber: number;
    status: string;
    reservedAt: Date;
  };
  onCancel: (id: string) => void;
}

export function ReservationCard({ reservation, onCancel }: Readonly<ReservationCardProps>) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Table {reservation.tableNumber}</CardTitle>
      </CardHeader>
      <CardContent>
        <p>Status: {reservation.status}</p>
        <p>Reserved: {reservation.reservedAt.toLocaleDateString()}</p>
        <Button onClick={() => onCancel(reservation.id)} variant="destructive">
          Cancel
        </Button>
      </CardContent>
    </Card>
  );
}
```

---

## Adding an API Route

### GET Route

```bash
# Create API route
mkdir -p app/api/reservations
touch app/api/reservations/route.ts
```

```typescript
// app/api/reservations/route.ts
import { NextRequest, NextResponse } from "next/server";
import { GetAllReservationsUseCase } from "@/src/application/cafeteria/use-cases/reservations/GetAllReservationsUseCase";

export async function GET(request: NextRequest) {
  try {
    const reservationRepository = new PrismaReservationRepository(prisma);
    const useCase = new GetAllReservationsUseCase(reservationRepository);
    const reservations = await useCase.execute();

    return NextResponse.json(reservations);
  } catch (error) {
    return NextResponse.json(
      { error: "Failed to fetch reservations" },
      { status: 500 }
    );
  }
}
```

### Dynamic Route

```bash
# Create dynamic API route
mkdir -p app/api/reservations/[id]
touch app/api/reservations/[id]/route.ts
```

```typescript
// app/api/reservations/[id]/route.ts
interface RouteContext {
  params: { id: string };
}

export async function GET(request: NextRequest, { params }: RouteContext) {
  try {
    const reservation = await fetchReservation(params.id);
    return NextResponse.json(reservation);
  } catch (error) {
    return NextResponse.json({ error: "Not found" }, { status: 404 });
  }
}

export async function DELETE(request: NextRequest, { params }: RouteContext) {
  try {
    await deleteReservation(params.id);
    return NextResponse.json({ success: true });
  } catch (error) {
    return NextResponse.json({ error: "Failed to delete" }, { status: 500 });
  }
}
```

---

## Adding shadcn/ui Components

### Install a Single Component

```bash
npx shadcn@latest add button
```

### Install Multiple Components

```bash
npx shadcn@latest add button card dialog input label
```

### Use the Component

```typescript
import { Button } from '@/app/components/ui/button';

<Button variant="default">Click me</Button>
<Button variant="destructive">Delete</Button>
<Button variant="outline">Cancel</Button>
<Button variant="ghost" size="sm">Small Button</Button>
```

### Available Variants and Sizes

**Button Variants:**

- `default` - Primary button
- `destructive` - Dangerous actions (delete, etc.)
- `outline` - Secondary action
- `secondary` - Alternative style
- `ghost` - Minimal style
- `link` - Link-styled button

**Button Sizes:**

- `default` - Normal size
- `sm` - Small
- `lg` - Large
- `icon` - Icon-only button

### Customize Components

```typescript
// app/components/ui/button.tsx
// You can edit the generated file to add custom variants

const buttonVariants = cva("inline-flex items-center justify-center...", {
  variants: {
    variant: {
      default: "bg-primary...",
      destructive: "bg-destructive...",
      // Add your custom variant
      custom: "bg-purple-500 text-white hover:bg-purple-600",
    },
  },
});
```

---

## Formatting Code with Prettier

### Format All Files

```bash
npm run lint:fix
```

This command:

1. Runs ESLint with `--fix` flag
2. Formats code with Prettier

### Format Specific File

```bash
npx prettier --write app/components/Button.tsx
```

### Format Directory

```bash
npx prettier --write "app/**/*.{ts,tsx}"
npx prettier --write "src/**/*.{ts,tsx}"
```

### Check Formatting (CI)

```bash
npx prettier --check "**/*.{ts,tsx}"
```

### IDE Integration

**VS Code:**

1. Install "Prettier - Code formatter" extension
2. Add to `.vscode/settings.json`:

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

## Database Operations

### Add Prisma Schema

```prisma
// src/infrastructure/persistence/prisma/schema.prisma
model Reservation {
  id          String   @id @default(cuid())
  customerId  String
  tableNumber Int
  status      String
  reservedAt  DateTime
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  customer Customer @relation(fields: [customerId], references: [id])

  @@index([customerId])
}
```

### Generate Migration

```bash
npx prisma migrate dev --name add_reservation_table
```

### Apply Migration

```bash
npx prisma migrate deploy
```

### Generate Prisma Client

```bash
npx prisma generate
```

### View Database

```bash
npx prisma studio
```

---

## Styling Tasks

### Add Global CSS Variables

```css
/* app/globals.css */
:root {
  --primary-color: #3b82f6;
  --secondary-color: #8b5cf6;
  --border-radius: 0.5rem;
}

@media (prefers-color-scheme: dark) {
  :root {
    --primary-color: #60a5fa;
    --secondary-color: #a78bfa;
  }
}
```

### Use CSS Variables

```typescript
<div className="bg-[var(--primary-color)]">
  Content
</div>
```

### Add Custom Tailwind Utilities

```typescript
// tailwind.config.ts
import type { Config } from "tailwindcss";

const config: Config = {
  theme: {
    extend: {
      colors: {
        "brand-blue": "#3b82f6",
        "brand-purple": "#8b5cf6",
      },
      spacing: {
        "18": "4.5rem",
        "88": "22rem",
      },
    },
  },
};

export default config;
```

---

## Troubleshooting

### Module Not Found

```bash
# Solution: Install dependencies
npm install

# If still not working, clear cache
rm -rf node_modules
rm package-lock.json
npm install
```

### TypeScript Errors

```bash
# Check TypeScript errors
npx tsc --noEmit

# Common fixes:
# 1. Check import paths
# 2. Verify tsconfig.json paths
# 3. Ensure types are exported
```

### Tailwind Not Applying

```bash
# 1. Check globals.css has @import "tailwindcss"
# 2. Restart dev server
npm run dev

# 3. Clear .next folder
rm -rf .next
npm run dev
```

### Port 3000 In Use

```bash
# Use different port
npm run dev -- -p 3001

# Or kill process on port 3000
lsof -ti:3000 | xargs kill -9
```

### ESLint Errors

```bash
# Auto-fix issues
npm run lint:fix

# View errors only
npm run lint
```

### Hot Reload Not Working

```bash
# 1. Restart dev server
# 2. Clear .next folder
rm -rf .next
npm run dev

# 3. Check file is in app/ directory
# 4. Verify no syntax errors
```

---

## Summary

### Quick Command Reference

```bash
# Development
npm run dev              # Start dev server
npm run build            # Build for production
npm run start            # Run production build

# Linting & Formatting
npm run lint             # Check for issues
npm run lint:fix         # Fix issues + format

# Database
npx prisma migrate dev   # Create migration
npx prisma generate      # Generate client
npx prisma studio        # View database

# shadcn/ui
npx shadcn@latest add <component>  # Add component

# Prettier
npx prettier --write <file>        # Format file
```

---

**Back to:** [CLAUDE.md](../../CLAUDE.md)
