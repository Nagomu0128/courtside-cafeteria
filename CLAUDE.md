# CLAUDE.md - AI Assistant Guide for Cafeteria Management System

> **Last Updated:** 2025-11-16
> **Project:** Cafeteria Management System
> **Status:** Initial Development Phase

---

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Tech Stack](#tech-stack)
3. [Codebase Structure](#codebase-structure)
4. [Development Workflow](#development-workflow)
5. [Key Conventions](#key-conventions)
6. [File Organization Rules](#file-organization-rules)
7. [Coding Standards](#coding-standards)
8. [Component Patterns](#component-patterns)
9. [Styling Guidelines](#styling-guidelines)
10. [Common Tasks](#common-tasks)
11. [Deployment](#deployment)
12. [Troubleshooting](#troubleshooting)

---

## 🎯 Project Overview

**Cafeteria Management System** is a modern web application built with Next.js for managing cafeteria operations. The project is currently in its initial development phase with the base template setup.

### Project Goals
- Provide an efficient cafeteria management solution
- Modern, responsive UI with dark mode support
- Type-safe development with TypeScript
- Fast, optimized production builds

### Current Status
- ✅ Base Next.js 16 template initialized
- ✅ TypeScript configuration complete
- ✅ Tailwind CSS v4 integrated
- ✅ ESLint configured with Next.js rules
- 🚧 Application features pending implementation

---

## 🛠 Tech Stack

### Core Framework
- **Next.js 16.0.3** - React framework with App Router
- **React 19.2.0** - Latest React with Server Components
- **TypeScript 5.x** - Type-safe JavaScript with strict mode

### Styling
- **Tailwind CSS v4** - Utility-first CSS framework (latest version)
- **PostCSS** - CSS processing with Tailwind plugin
- **Geist Font Family** - Vercel's optimized font (sans & mono)

### Development Tools
- **ESLint 9.x** - Linting with flat config format
- **Next.js ESLint Config** - Core Web Vitals rules
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

## 📁 Codebase Structure

```
cafeteria-mg/
├── app/                          # Next.js App Router directory
│   ├── layout.tsx               # Root layout with metadata & fonts
│   ├── page.tsx                 # Home page component
│   ├── globals.css              # Global styles & Tailwind imports
│   └── favicon.ico              # App favicon
│
├── public/                       # Static assets (served at /)
│   ├── next.svg                 # Next.js logo
│   ├── vercel.svg               # Vercel logo
│   └── *.svg                    # Various icons
│
├── Configuration Files
├── .gitignore                   # Git ignore patterns
├── eslint.config.mjs            # ESLint flat config
├── next.config.ts               # Next.js configuration
├── package.json                 # NPM dependencies & scripts
├── postcss.config.mjs           # PostCSS with Tailwind
├── tsconfig.json                # TypeScript compiler config
├── README.md                    # Standard Next.js readme
└── CLAUDE.md                    # This file
```

### Directory Purposes

| Directory/File | Purpose | Rules |
|---------------|---------|-------|
| `/app` | Application source code (App Router) | All routes, layouts, and pages |
| `/app/layout.tsx` | Root layout component | Defines HTML structure, fonts, metadata |
| `/app/page.tsx` | Home page route | Default export function component |
| `/app/globals.css` | Global CSS styles | Tailwind imports, CSS variables, base styles |
| `/public` | Static assets | Accessible at root URL path |
| `/node_modules` | Dependencies | Auto-generated, never commit |
| `/.next` | Build output | Auto-generated, never commit |

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

### Component Organization

**When creating new components:**

1. **Shared Components** - Create in `/app/components/` (future directory)
   ```
   app/components/
   ├── Button.tsx
   ├── Card.tsx
   └── Input.tsx
   ```

2. **Route-Specific Components** - Co-locate with route
   ```
   app/dashboard/
   ├── components/
   │   └── DashboardCard.tsx
   └── page.tsx
   ```

3. **Component Structure** - Each complex component can have:
   ```
   components/UserProfile/
   ├── UserProfile.tsx      # Main component
   ├── UserProfile.types.ts # TypeScript types
   ├── UserProfile.test.tsx # Tests (future)
   └── index.ts             # Re-export
   ```

### Route Organization (App Router)

```
app/
├── (auth)/              # Route group (doesn't affect URL)
│   ├── login/
│   │   └── page.tsx    # /login
│   └── register/
│       └── page.tsx    # /register
├── dashboard/
│   ├── layout.tsx      # Dashboard layout
│   ├── page.tsx        # /dashboard
│   └── settings/
│       └── page.tsx    # /dashboard/settings
└── api/                 # API routes (future)
    └── users/
        └── route.ts     # /api/users
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

---

## 💻 Coding Standards

### TypeScript Standards

✅ **DO:**
- Enable all strict mode checks
- Use explicit types for function parameters
- Define interfaces for object shapes
- Use `unknown` instead of `any` when type is truly unknown
- Leverage type inference where obvious

❌ **DON'T:**
- Use `any` type (use `unknown` or proper types)
- Disable TypeScript errors with `@ts-ignore`
- Use non-null assertions (`!`) without justification
- Leave unused variables or imports

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
3. **Consistent Formatting** - Let ESLint handle it
4. **Clear Naming** - Use descriptive, unambiguous names
5. **Comments** - Explain "why", not "what"

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

### When Making Changes

1. **Always read files before editing** - Use Read tool first
2. **Follow existing patterns** - Match the codebase style
3. **Use TypeScript strictly** - No `any` types
4. **Test locally** - Run `npm run dev` to verify changes
5. **Run lint** - Execute `npm run lint` before committing
6. **Commit atomically** - One logical change per commit

### Code Generation Guidelines

1. **Components**: Use functional components with TypeScript
2. **Styling**: Prefer Tailwind utilities over custom CSS
3. **Server vs Client**: Default to Server Components, use "use client" only when necessary
4. **Imports**: Use `@/` alias for clean imports
5. **Metadata**: Always export metadata for pages
6. **Error Handling**: Add proper error boundaries and fallbacks (future)

### Best Practices for AI Collaboration

- **Ask for clarification** if requirements are ambiguous
- **Suggest alternatives** when better patterns exist
- **Document complex logic** with clear comments
- **Follow existing conventions** rather than introducing new patterns
- **Consider performance** - Server Components over Client when possible
- **Think about accessibility** - semantic HTML, ARIA labels
- **Mobile-first design** - responsive by default

### What Not to Do

- Don't create files in root directory unnecessarily
- Don't add dependencies without discussing
- Don't disable TypeScript checks
- Don't ignore ESLint errors
- Don't write custom CSS when Tailwind utility exists
- Don't commit `node_modules`, `.next`, or `.env` files

---

## 📚 Additional Resources

### Official Documentation

- [Next.js 16 Docs](https://nextjs.org/docs)
- [React 19 Docs](https://react.dev)
- [TypeScript Docs](https://www.typescriptlang.org/docs/)
- [Tailwind CSS v4 Docs](https://tailwindcss.com/docs)

### Learning Resources

- [Next.js Learn](https://nextjs.org/learn) - Interactive tutorial
- [React Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)
- [App Router Migration](https://nextjs.org/docs/app/building-your-application/upgrading/app-router-migration)

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

**Last Updated:** 2025-11-16
**Maintained By:** AI Assistants working on cafeteria-mg
**Version:** 1.0.0

---

*This document should be updated whenever significant architectural decisions are made or new patterns are established.*
