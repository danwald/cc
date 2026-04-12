# TypeScript / Frontend Project Setup

Standards and tooling conventions for TypeScript and React/Next.js projects.

## Core Tools

| Tool | Purpose | Version |
|------|---------|---------|
| Node.js | Runtime | 20+ LTS |
| npm / pnpm | Package manager | pnpm 9.x or npm |
| TypeScript | Language | 5.x (strict) |
| ESLint | Linting | 9.x flat config |
| Prettier | Formatting | 3.x |
| Vitest | Unit/component testing | latest |
| Playwright | E2E testing | latest |

## Scaffolding a New Next.js Project

```bash
npx create-next-app@latest frontend \
  --typescript --tailwind --eslint --app --src-dir \
  --import-alias "@/*" --use-npm
```

Then install additional dependencies:

```bash
cd frontend && npm install clsx tailwind-merge zod
npm install -D vitest @vitejs/plugin-react @testing-library/react \
  @testing-library/dom @testing-library/user-event @testing-library/jest-dom \
  vite-tsconfig-paths jsdom prettier prettier-plugin-tailwindcss
```

## Project Structure

```
frontend/src/
├── app/                    # Next.js App Router (routing only — keep thin)
│   ├── layout.tsx          # Root layout (fonts, providers)
│   ├── page.tsx            # Landing page (Server Component)
│   ├── loading.tsx         # Root loading skeleton
│   ├── error.tsx           # Root error boundary ("use client")
│   ├── not-found.tsx       # 404 page
│   └── global-error.tsx    # Catches root layout errors
├── components/
│   ├── ui/                 # Reusable primitives (Button, Input, Card)
│   ├── layout/             # Shell components (Header, Footer, Sidebar)
│   └── features/           # Domain-specific composites
├── hooks/                  # Custom React hooks
├── lib/
│   ├── utils.ts            # cn() and other helpers
│   └── api.ts              # Type-safe fetch client
├── types/
│   └── index.ts            # Shared TypeScript types
└── styles/
    └── globals.css
```

## tsconfig.json (strict)

```json
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "preserve",
    "incremental": true
  }
}
```

## TypeScript Conventions

- **No `any`**: use `unknown` and narrow with type guards. Use `Record<string, unknown>` for generic objects.
- **Prefer `type` over `interface`** for props, unions, intersections. Use `interface` only for declaration merging or `implements`.
- **Export prop types** alongside their components.
- Strict mode + `noUncheckedIndexedAccess` catches most footguns at compile time.

## React / Next.js Patterns

### Component Authoring
- Default to **Server Components**. Only add `"use client"` when you need state, effects, event handlers, or browser APIs.
- Push `"use client"` as deep in the tree as possible — extract small interactive pieces into focused Client Components.
- **One component per file** — kebab-case filenames (`user-profile.tsx`), PascalCase exports (`UserProfile`).
- Co-locate tests, styles, and sub-components next to their component.

### Next.js App Router
- Route groups `(marketing)/`, `(app)/` for separate layouts without URL impact.
- Private folders `_components/` for co-located helpers excluded from routing.
- Special files: `layout.tsx`, `loading.tsx`, `error.tsx` (must be `"use client"`), `not-found.tsx`, `global-error.tsx`.
- **Server Actions**: validate with Zod, re-verify auth inside every action, return errors as values (don't throw).
- **Metadata API**: use static `metadata` export or `generateMetadata()` for dynamic SEO.

### State Management

| Concern | Approach |
|---------|----------|
| Server state | Fetch in Server Components; TanStack Query in Client Components |
| Global client state | Zustand (if needed) |
| Form state | React Hook Form + Zod, or `useActionState` with Server Actions |
| URL state | `nuqs` for search params as state |
| Shared context | React Context — only for low-frequency updates (theme, locale, auth) |

Never use React Context for high-frequency state updates.

### Styling (Tailwind CSS v4)
- CSS-first config via `@theme` directive (no `tailwind.config.js`).
- Create `cn()` utility with `clsx` + `tailwind-merge` in `src/lib/utils.ts`.
- Use `prettier-plugin-tailwindcss` to auto-sort class names.
- Avoid `@apply` — write utility classes directly or extract components.

**`src/lib/utils.ts`:**

```typescript
import { type ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

### Performance
- `next/image` with `priority` for LCP images; always provide `sizes` for responsive images.
- `next/font/google` with `variable` option for self-hosted fonts with zero layout shift.
- `dynamic()` with `ssr: false` for browser-only heavy components.
- **React Compiler** (Next.js 16+): `useMemo`/`useCallback`/`React.memo` are automatic — don't add them manually.

**`src/lib/api.ts`** — type-safe fetch client:

```typescript
const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:8000"

export async function fetchApi<T>(endpoint: string, options?: RequestInit): Promise<T> {
  const res = await fetch(`${API_BASE_URL}${endpoint}`, {
    headers: { "Content-Type": "application/json", ...options?.headers },
    ...options,
  })
  if (!res.ok) throw new Error(`API error: ${res.status}`)
  return res.json() as Promise<T>
}
```

**`src/app/layout.tsx`** — root layout with `next/font/google`:

```tsx
import type { Metadata } from "next"
import { Geist, Geist_Mono } from "next/font/google"
import "@/styles/globals.css"

const geist = Geist({ subsets: ["latin"], variable: "--font-sans" })
const geistMono = Geist_Mono({ subsets: ["latin"], variable: "--font-mono" })

export const metadata: Metadata = { title: "My App", description: "..." }

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${geist.variable} ${geistMono.variable}`}>
      <body className="font-sans antialiased">{children}</body>
    </html>
  )
}
```

**`src/app/error.tsx`** — route error boundary (must be `"use client"`):

```tsx
"use client"

import { useEffect } from "react"

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => { console.error(error) }, [error])
  return (
    <div className="flex min-h-screen flex-col items-center justify-center gap-4">
      <h2 className="text-xl font-semibold">Something went wrong</h2>
      <button onClick={reset} className="rounded-md bg-blue-600 px-4 py-2 text-white">
        Try again
      </button>
    </div>
  )
}
```

`global-error.tsx` at app root must render its own `<html>` and `<body>` tags and be `"use client"`.

### Security
- `server-only` import on modules that must never reach the client.
- Security headers in `next.config.ts`:

```typescript
const nextConfig = {
  async headers() {
    return [{
      source: "/(.*)",
      headers: [
        { key: "X-Frame-Options", value: "DENY" },
        { key: "X-Content-Type-Options", value: "nosniff" },
        { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
        { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
      ],
    }]
  },
}
```

- Data Access Layer: always verify auth before returning data; return only safe fields.

### Error Handling
- `error.tsx` per route segment (must be `"use client"`) — catches errors in `page.tsx` and children.
- `global-error.tsx` at app root (must render its own `<html>` and `<body>`).
- Server Actions: return errors as values with Zod validation, don't throw.
- API routes: `try/catch` with structured error responses.

## Linting & Formatting

### ESLint (flat config)

```js
// eslint.config.mjs
import { FlatCompat } from "@eslint/eslintrc"

const compat = new FlatCompat()

export default [
  ...compat.extends(
    "next/core-web-vitals",
    "next/typescript",
    "plugin:@typescript-eslint/strict-type-checked",
    "plugin:jsx-a11y/recommended",
    "prettier"  // always last
  ),
]
```

### Prettier

```json
{
  "semi": false,
  "singleQuote": false,
  "tabWidth": 2,
  "trailingComma": "all",
  "printWidth": 100,
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

### Pre-commit (husky + lint-staged)

```json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{js,json,css,md}": ["prettier --write"]
  }
}
```

## Testing

### Vitest (unit/component)

```typescript
// vitest.config.mts
import react from "@vitejs/plugin-react"
import tsconfigPaths from "vite-tsconfig-paths"
import { defineConfig } from "vitest/config"

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./src/test/setup.ts"],
    include: ["src/**/*.test.{ts,tsx}"],
    coverage: {
      thresholds: { statements: 80, functions: 80, lines: 80, branches: 75 },
    },
  },
})
```

**`src/test/setup.ts`:**

```typescript
import "@testing-library/jest-dom/vitest"
```

- Co-locate unit/component tests as `*.test.tsx` next to the component.
- Use React Testing Library (`@testing-library/react`).
- Use MSW (Mock Service Worker) for API mocking.

### Playwright (E2E)

```typescript
// playwright.config.ts
import { defineConfig } from "@playwright/test"

export default defineConfig({
  testDir: "./tests/e2e",
  use: { baseURL: "http://localhost:3000" },
})
```

Keep E2E tests in `tests/e2e/*.spec.ts`. Test happy paths only.

## CI (GitHub Actions)

```yaml
jobs:
  quality:
    steps:
      - uses: actions/setup-node@v4
        with: { node-version: "20" }
      - run: npm ci
      - run: npm run format -- --check
      - run: npm run lint
      - run: npx tsc --noEmit
      - run: npm run test:coverage
      - uses: codecov/codecov-action@v4
```

Enable Dependabot for weekly `npm` dependency updates.

## package.json Scripts

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "lint": "next lint",
    "format": "prettier --write \"src/**/*.{ts,tsx,css}\"",
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "typecheck": "tsc --noEmit"
  }
}
```
