# Next.js Enterprise Project Structure

> A production-ready, scalable project architecture for Next.js applications using the App Router.

---

## Full Directory Tree

```
my-enterprise-app/
├── src/
│   ├── app/
│   │   ├── layout.tsx                  # Root layout — providers, fonts, metadata, lang attribute
│   │   ├── not-found.tsx               # Custom 404 page
│   │   ├── error.tsx                   # Route-level error boundary
│   │   ├── global-error.tsx            # Root layout error boundary (catches layout errors)
│   │   ├── forbidden.tsx               # Custom 403 page (experimental: authInterrupts)
│   │   ├── unauthorized.tsx            # Custom 401 page (experimental: authInterrupts)
│   │   │
│   │   ├── api/
│   │   │   ├── health/
│   │   │   │   └── route.ts            # Health check endpoint
│   │   │   ├── auth/
│   │   │   │   └── [...nextauth]/
│   │   │   │       └── route.ts        # Auth.js v5 route handler
│   │   │   └── webhooks/
│   │   │       ├── stripe/route.ts     # Stripe webhook handler
│   │   │       └── github/route.ts     # GitHub webhook handler
│   │   │
│   │   └── [locale]/                   # Dynamic locale segment (en, fr, de, ar, ja, ...)
│   │       ├── layout.tsx              # Locale layout — sets dir, loads translations
│   │       ├── page.tsx                # Home page (/en, /fr, ...)
│   │       ├── loading.tsx             # Global loading UI (Suspense)
│   │       │
│   │       ├── (auth)/
│   │       │   ├── layout.tsx          # Auth layout (centered card)
│   │       │   ├── login/
│   │       │   │   └── page.tsx        # /[locale]/login
│   │       │   └── register/
│   │       │       └── page.tsx        # /[locale]/register
│   │       │
│   │       ├── (shop)/
│   │       │   ├── layout.tsx          # Shop layout (navbar + cart + footer)
│   │       │   ├── products/
│   │       │   │   ├── page.tsx        # /[locale]/products (product listing)
│   │       │   │   ├── loading.tsx     # Skeleton grid while loading
│   │       │   │   └── [slug]/
│   │       │   │       ├── page.tsx    # /[locale]/products/:slug (product detail)
│   │       │   │       └── loading.tsx # Skeleton detail while loading
│   │       │   ├── cart/
│   │       │   │   └── page.tsx        # /[locale]/cart
│   │       │   └── checkout/
│   │       │       └── page.tsx        # /[locale]/checkout
│   │       │
│   │       └── (dashboard)/
│   │           ├── layout.tsx          # Dashboard layout (sidebar + header)
│   │           ├── overview/
│   │           │   └── page.tsx        # /[locale]/overview
│   │           ├── settings/
│   │           │   └── page.tsx        # /[locale]/settings
│   │           └── users/
│   │               ├── page.tsx        # /[locale]/users (list)
│   │               └── [id]/
│   │                   └── page.tsx    # /[locale]/users/:id (detail)
│   │
│   ├── actions/                        # Server Actions ("use server")
│   │   ├── user.actions.ts             # User create, update, delete actions
│   │   ├── auth.actions.ts             # Login, register, logout actions
│   │   └── cart.actions.ts             # Add to cart, remove, update quantity
│   │
│   ├── components/
│   │   ├── ui/                         # Primitive components (Button, Input, Modal)
│   │   │   ├── button.tsx
│   │   │   ├── button.test.tsx         # Button variants, sizes, disabled
│   │   │   ├── input.tsx
│   │   │   ├── modal.tsx
│   │   │   ├── badge.tsx
│   │   │   ├── card.tsx
│   │   │   ├── tooltip.tsx
│   │   │   ├── toaster.tsx
│   │   │   └── index.ts                # Barrel export
│   │   ├── layout/                     # Layout components
│   │   │   ├── sidebar.tsx
│   │   │   ├── header.tsx
│   │   │   ├── footer.tsx
│   │   │   └── page-container.tsx
│   │   ├── auth/                       # Auth-related components
│   │   │   └── auth-guard.tsx          # Route protection wrapper
│   │   ├── product/                    # Product components + co-located tests
│   │   │   ├── product-card.tsx        # Card for product grid
│   │   │   ├── product-card.test.tsx   # Rendering, badges, links
│   │   │   ├── product-grid.tsx        # Responsive product grid
│   │   │   ├── product-grid.test.tsx   # Grid layout, empty state
│   │   │   ├── product-gallery.tsx     # Image gallery with thumbnails
│   │   │   ├── product-info.tsx        # Price, title, description
│   │   │   ├── product-filters.tsx     # Category, price range, sort
│   │   │   ├── product-filters.test.tsx # Filter interactions, URL updates
│   │   │   ├── add-to-cart-button.tsx  # Add to cart with quantity
│   │   │   └── add-to-cart-button.test.tsx # Quantity, mutation, disabled
│   │   ├── cart/                       # Cart components + co-located tests
│   │   │   ├── cart-item.tsx           # Single cart line item
│   │   │   ├── cart-summary.tsx        # Subtotal, tax, total
│   │   │   ├── cart-summary.test.tsx   # Subtotal, tax, total display
│   │   │   └── cart-icon.tsx           # Navbar cart icon with count
│   │   ├── forms/                      # Form components (useActionState + Zod)
│   │   │   ├── user-form.tsx
│   │   │   ├── login-form.tsx
│   │   │   └── settings-form.tsx
│   │   └── providers.tsx               # All context providers composed together
│   │
│   ├── data-layer/                     # Backend-agnostic data layer (Adapter + Repository pattern)
│   │   ├── types/                      # Canonical domain types — independent of any backend
│   │   │   ├── product.ts              # Product, ProductImage, ProductVariant, ProductFilters
│   │   │   ├── cart.ts                 # Cart, CartItem, CartSummary
│   │   │   ├── user.ts                 # User, UserProfile, UserRole
│   │   │   └── order.ts               # Order, OrderItem, OrderStatus
│   │   ├── interfaces/                 # Repository contracts — every adapter must implement these
│   │   │   ├── product.repository.ts   # IProductRepository
│   │   │   ├── cart.repository.ts      # ICartRepository
│   │   │   ├── user.repository.ts      # IUserRepository
│   │   │   └── order.repository.ts     # IOrderRepository
│   │   ├── adapters/
│   │   │   ├── magento/                # Magento 2 GraphQL adapter
│   │   │   │   ├── client.ts           # graphql-request GraphQLClient for Magento
│   │   │   │   ├── product.adapter.ts  # Implements IProductRepository (queries inline via gql.tada)
│   │   │   │   ├── cart.adapter.ts     # Implements ICartRepository
│   │   │   │   ├── user.adapter.ts     # Implements IUserRepository
│   │   │   │   └── index.ts            # Exports MagentoAdapter
│   │   │   ├── shopify/                # Shopify Storefront API adapter (GraphQL)
│   │   │   │   ├── client.ts           # graphql-request GraphQLClient with Storefront auth
│   │   │   │   ├── product.adapter.ts
│   │   │   │   ├── cart.adapter.ts
│   │   │   │   ├── user.adapter.ts
│   │   │   │   └── index.ts            # Exports ShopifyAdapter
│   │   │   ├── odoo/                   # Odoo REST API adapter
│   │   │   │   ├── client.ts           # Authenticated fetch wrapper for Odoo REST
│   │   │   │   ├── product.adapter.ts
│   │   │   │   ├── cart.adapter.ts
│   │   │   │   ├── user.adapter.ts
│   │   │   │   └── index.ts            # Exports OdooAdapter
│   │   │   └── custom/                 # Template — copy this to add any new backend
│   │   │       ├── client.ts           # HTTP client setup (REST or GraphQL)
│   │   │       ├── product.adapter.ts  # Stub — implement IProductRepository
│   │   │       ├── cart.adapter.ts     # Stub — implement ICartRepository
│   │   │       ├── user.adapter.ts     # Stub — implement IUserRepository
│   │   │       └── index.ts            # Exports CustomAdapter
│   │   └── factory.ts                  # Resolves active adapter from BACKEND_PROVIDER env var
│   │
│   ├── i18n/
│   │   ├── config.ts                    # Supported locales, default locale, fallback rules
│   │   ├── get-translations.ts          # Server-side: load translations for a given locale
│   │   ├── client-provider.tsx          # "use client" — i18n context provider (IntlProvider)
│   │   ├── navigation.ts               # Locale-aware Link, redirect, usePathname wrappers
│   │   └── locales/
│   │       ├── en/
│   │       │   ├── common.json          # Shared strings (nav, buttons, errors, footer)
│   │       │   ├── products.json        # Product page strings (filters, badges, labels)
│   │       │   ├── cart.json            # Cart & checkout strings
│   │       │   └── dashboard.json       # Dashboard strings
│   │       ├── fr/
│   │       │   ├── common.json
│   │       │   ├── products.json
│   │       │   ├── cart.json
│   │       │   └── dashboard.json
│   │       ├── de/
│   │       │   ├── common.json
│   │       │   ├── products.json
│   │       │   ├── cart.json
│   │       │   └── dashboard.json
│   │       └── ar/
│   │           ├── common.json          # RTL language support
│   │           ├── products.json
│   │           ├── cart.json
│   │           └── dashboard.json
│   │
│   ├── lib/
│   │   ├── utils.ts                    # Pure utilities — cn(), formatDate()
│   │   ├── utils.test.ts              # cn(), formatDate() tests
│   │   ├── currency.ts                # formatPrice() with locale + currency, Intl.NumberFormat
│   │   ├── currency.test.ts           # Multi-currency formatting tests (USD, EUR, JPY, etc.)
│   │   ├── api-client.ts              # Type-safe fetch wrapper with auth, retries, store headers
│   │   ├── auth.ts                    # Auth.js v5 config — exports handlers, auth, signIn, signOut
│   │   ├── metadata.ts                # generatePageMetadata() — locale-aware titles, alternates, openGraph
│   │   ├── web-vitals.ts              # reportWebVitals() — CLS, LCP, INP reporting to analytics
│   │   └── validations/
│   │       ├── user.ts               # User schemas (create, update)
│   │       ├── auth.ts               # Auth schemas (login, register)
│   │       └── common.ts             # Shared schemas (pagination, search)
│   │
│   ├── hooks/
│   │   ├── use-debounce.ts
│   │   ├── use-debounce.test.ts        # Debounce timing with fake timers
│   │   ├── use-media-query.ts
│   │   ├── use-auth.ts
│   │   ├── use-translations.ts         # Client-side translation hook — useTranslations(namespace)
│   │   ├── use-currency.ts             # Currency formatting hook — useCurrency()
│   │   ├── use-store-config.ts         # Current store config hook — useStoreConfig()
│   │   ├── use-local-storage.ts
│   │   └── use-local-storage.test.ts   # Read/write/clear localStorage
│   │
│   ├── services/
│   │   ├── user.service.ts            # User queries via backend adapter (provider-agnostic)
│   │   ├── user.service.test.ts       # User service unit test
│   │   ├── billing.service.ts         # Stripe integration (multi-currency aware)
│   │   ├── currency.service.ts        # Exchange rate fetching, caching, conversion
│   │   ├── currency.service.test.ts   # Exchange rate & conversion tests
│   │   └── notification.service.ts    # Email, push, in-app notifications
│   │
│   ├── stores/
│   │   ├── ui.store.ts                # Sidebar, theme, modals (Zustand)
│   │   ├── ui.store.test.ts           # Sidebar toggle, theme switching
│   │   ├── cart.store.ts              # Shopping cart state — currency-aware (Zustand + persist)
│   │   ├── cart.store.test.ts         # Cart ID persistence
│   │   ├── currency.store.ts          # Active currency, exchange rates (Zustand + persist)
│   │   ├── currency.store.test.ts     # Currency switching, rate caching tests
│   │   ├── store-config.store.ts      # Active store/tenant context (Zustand)
│   │   └── notification.store.ts      # Toast/notification queue
│   │
│   ├── types/
│   │   ├── api.ts                     # ApiResponse<T>, ApiError
│   │   ├── models.ts                  # User, Team, Project interfaces
│   │   ├── i18n.ts                    # Locale, TranslationKey, Messages types
│   │   ├── currency.ts               # Currency, ExchangeRate, MoneyAmount types
│   │   ├── store.ts                   # StoreConfig, StoreTheme types
│   │   └── globals.d.ts              # Global type augmentations
│   │
│   ├── styles/
│   │   └── globals.css                # @import "tailwindcss", @theme config, keyframes
│   │
│   ├── config/
│   │   ├── site.ts                    # App name, description, URLs, preconnect origins
│   │   ├── nav.ts                     # Navigation items (per-store overrides)
│   │   ├── env.ts                     # Validated environment variables (via Zod)
│   │   ├── currencies.ts             # Supported currencies, symbols, decimal rules
│   │   └── stores.ts                 # Multi-store definitions (region, domain, locale, currency, theme)
│   │
│   ├── constants/
│   │   ├── roles.ts                   # USER_ROLES = ["admin", "member", "viewer"]
│   │   └── limits.ts                  # MAX_UPLOAD_SIZE, PAGINATION_LIMIT
│   │
│   └── proxy.ts                       # Proxy — auth redirects, headers (verify sessions in Server Components too)
│
├── public/
│   ├── images/
│   ├── fonts/
│   ├── favicon.ico
│   ├── robots.txt
│   └── sitemap.xml
│
├── tests/
│   ├── setup.ts                       # Test setup — imports @testing-library/jest-dom
│   ├── e2e/                           # Playwright — critical user journeys
│   │   ├── auth.spec.ts
│   │   ├── products.spec.ts
│   │   ├── cart.spec.ts
│   │   └── dashboard.spec.ts
│   └── fixtures/
│       ├── users.ts                   # Factory functions for test data
│       ├── products.ts                # Product test data factories
│       └── teams.ts
│
├── docs/
│   ├── ARCHITECTURE.md                # Architecture Decision Records
│   ├── API.md                         # API documentation
│   └── ONBOARDING.md                 # New developer guide
│
├── scripts/
│   └── migrate.ts                     # Data migration helpers
│
├── docker/
│   ├── Dockerfile                     # Multi-stage production build
│   └── docker-compose.yml            # Local dev (app + Redis)
│
├── .github/
│   └── workflows/
│       ├── ci.yml                     # Lint → Type-check → Test → Build
│       └── deploy.yml                 # Production deployment
│
├── next.config.ts
├── vitest.config.ts                   # Vitest configuration
├── tsconfig.json
├── eslint.config.mjs                  # ESLint 9 flat config
├── .prettierrc
├── .env.local                         # Local secrets (git-ignored)
├── .env.example                       # Template for new developers
├── .gitignore
├── package.json
└── README.md
```

---

## Key Code Examples

### Root Layout — `src/app/layout.tsx`

```tsx
import { Metadata } from "next";
import { Inter } from "next/font/google";
import "@/styles/globals.css";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: { default: "MyApp", template: "%s | MyApp" },
  description: "Enterprise application",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html suppressHydrationWarning>
      <body className={inter.className}>{children}</body>
    </html>
  );
}
```

### Locale Layout — `src/app/[locale]/layout.tsx`

> Sets the `lang` and `dir` attributes per locale. Loads translations and wraps children with i18n + store providers.

```tsx
import { notFound } from "next/navigation";
import { getTranslations } from "@/i18n/get-translations";
import { I18nProvider } from "@/i18n/client-provider";
import { Providers } from "@/components/providers";
import { SUPPORTED_LOCALES, type Locale } from "@/i18n/config";
import { getStoreByLocale } from "@/config/stores";

interface Props {
  children: React.ReactNode;
  params: Promise<{ locale: string }>;
}

export function generateStaticParams() {
  return SUPPORTED_LOCALES.map((locale) => ({ locale }));
}

export default async function LocaleLayout({ children, params }: Props) {
  const { locale } = await params;
  if (!SUPPORTED_LOCALES.includes(locale as Locale)) notFound();

  const messages = await getTranslations(locale as Locale);
  const store = getStoreByLocale(locale as Locale);
  const dir = ["ar", "he"].includes(locale) ? "rtl" : "ltr";

  return (
    <html lang={locale} dir={dir} suppressHydrationWarning>
      <body>
        <I18nProvider locale={locale} messages={messages}>
          <Providers storeConfig={store}>{children}</Providers>
        </I18nProvider>
      </body>
    </html>
  );
}
```

### Error Boundary — `src/app/error.tsx`

```tsx
"use client";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

### Global Error Boundary — `src/app/global-error.tsx`

> Catches errors in the root layout itself — `error.tsx` cannot catch root layout errors.

```tsx
"use client";

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <html>
      <body>
        <h2>Something went wrong!</h2>
        <button onClick={() => reset()}>Try again</button>
      </body>
    </html>
  );
}
```

### API Health Check — `src/app/api/health/route.ts`

```ts
import { NextResponse } from "next/server";

export async function GET() {
  return NextResponse.json({
    status: "healthy",
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION,
  });
}
```

### Dashboard Layout — `src/app/(dashboard)/layout.tsx`

> **Important:** Do not rely solely on the proxy for auth. Always verify sessions in Server Components and Route Handlers too (see [CVE-2025-29927](https://github.com/vercel/next.js/security/advisories/GHSA-f82v-jwr5-cevq)).

```tsx
import { redirect } from "next/navigation";
import { auth } from "@/lib/auth";
import { Sidebar } from "@/components/layout/sidebar";
import { Header } from "@/components/layout/header";

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const session = await auth();
  if (!session) redirect("/login");

  return (
    <div className="flex h-screen">
      <Sidebar />
      <div className="flex-1 flex flex-col">
        <Header />
        <main className="flex-1 p-6 overflow-auto">
          {children}
        </main>
      </div>
    </div>
  );
}
```

### Button Component with Variants — `src/components/ui/button.tsx`

```tsx
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "@/lib/utils";

const buttonVariants = cva(
  "inline-flex items-center rounded-md font-medium transition-colors",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground",
        outline: "border border-input bg-background hover:bg-accent",
        ghost: "hover:bg-accent hover:text-accent-foreground",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
);

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

export function Button({ className, variant, size, ...props }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  );
}
```

### Providers Composition — `src/components/providers.tsx`

> Composes all client-side providers: Auth, Theme, Currency, Store Config, and Toast notifications. No GraphQL provider here — the backend adapter is resolved server-side via `getBackend()`, so no client context is needed.

```tsx
"use client";

import { SessionProvider } from "next-auth/react";
import { ThemeProvider } from "next-themes";
import { Toaster } from "@/components/ui/toaster";
import { CurrencyProvider } from "@/stores/currency.store";
import { StoreConfigProvider } from "@/stores/store-config.store";
import type { StoreConfig } from "@/types/store";

interface ProvidersProps {
  children: React.ReactNode;
  storeConfig: StoreConfig;
}

export function Providers({ children, storeConfig }: ProvidersProps) {
  return (
    <SessionProvider>
      <StoreConfigProvider config={storeConfig}>
        <CurrencyProvider defaultCurrency={storeConfig.defaultCurrency}>
          <ThemeProvider attribute="class" defaultTheme="system">
            {children}
            <Toaster />
          </ThemeProvider>
        </CurrencyProvider>
      </StoreConfigProvider>
    </SessionProvider>
  );
}
```

### Utility Functions — `src/lib/utils.ts`

```ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

export function formatDate(date: Date, locale = "en-US"): string {
  return new Intl.DateTimeFormat(locale, {
    month: "short",
    day: "numeric",
    year: "numeric",
  }).format(date);
}

export function sleep(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

### Currency Formatting — `src/lib/currency.ts`

> Locale-aware currency formatting using `Intl.NumberFormat`. Supports all ISO 4217 currencies.

```ts
import type { Currency } from "@/types/currency";

export function formatPrice(
  amount: number,
  currency: Currency = "USD",
  locale = "en-US"
): string {
  return new Intl.NumberFormat(locale, {
    style: "currency",
    currency,
    minimumFractionDigits: getDecimalPlaces(currency),
    maximumFractionDigits: getDecimalPlaces(currency),
  }).format(amount);
}

function getDecimalPlaces(currency: Currency): number {
  const zeroDecimal: Currency[] = ["JPY", "KRW", "VND"];
  return zeroDecimal.includes(currency) ? 0 : 2;
}

export function convertPrice(
  amount: number,
  fromCurrency: Currency,
  toCurrency: Currency,
  rates: Record<string, number>
): number {
  if (fromCurrency === toCurrency) return amount;
  const baseAmount = amount / (rates[fromCurrency] ?? 1);
  return baseAmount * (rates[toCurrency] ?? 1);
}
```

### Zod Validation Schemas — `src/lib/validations/user.ts`

```ts
import { z } from "zod";

export const createUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  role: z.enum(["admin", "member", "viewer"]),
});

export const updateUserSchema = createUserSchema.partial();

export type CreateUserInput = z.infer<typeof createUserSchema>;
export type UpdateUserInput = z.infer<typeof updateUserSchema>;
```

### Service Layer — `src/services/user.service.ts`

> Services call repository interfaces via `getBackend()`. They never import Apollo, fetch, or any backend-specific code directly. Swapping backends requires zero changes here.

```ts
import { getBackend } from "@/data-layer/factory";
import { cache } from "react";

export const getUserById = cache(async (id: string) => {
  const { users } = getBackend();
  return users.getUserById(id);
});

export async function getUsers(page = 1, limit = 20) {
  const { users } = getBackend();
  return users.getUsers({ page, limit });
}

export async function createUser(input: CreateUserInput) {
  const { users } = getBackend();
  return users.createUser(input);
}
```

### Zustand Store — `src/stores/ui.store.ts`

```ts
import { create } from "zustand";

interface UIState {
  sidebarOpen: boolean;
  toggleSidebar: () => void;
  theme: "light" | "dark" | "system";
  setTheme: (theme: UIState["theme"]) => void;
}

export const useUIStore = create<UIState>((set) => ({
  sidebarOpen: true,
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
  theme: "system",
  setTheme: (theme) => set({ theme }),
}));
```

### Custom Hook — `src/hooks/use-debounce.ts`

```ts
import { useState, useEffect } from "react";

export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}
```

### Type Definitions — `src/types/api.ts`

```ts
export interface ApiResponse<T> {
  data: T;
  meta: { total: number; page: number; limit: number };
}

export interface ApiError {
  code: string;
  message: string;
  details?: Record<string, string[]>;
}
```

### Proxy — `src/proxy.ts`

> **Next.js 16:** `middleware.ts` is deprecated and renamed to `proxy.ts`. The proxy handles locale detection, store resolution, and auth redirects. Runs on **Node.js runtime only**.

```ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";
import { SUPPORTED_LOCALES, DEFAULT_LOCALE } from "@/i18n/config";
import { resolveStore } from "@/config/stores";

const publicPaths = ["/login", "/register", "/api/health"];

export function proxy(request: NextRequest) {
  const { pathname, hostname } = request.nextUrl;

  // 1. Resolve store from hostname (us.myapp.com → US store, eu.myapp.com → EU store)
  const store = resolveStore(hostname);
  const response = NextResponse.next();
  response.headers.set("x-store-id", store.id);

  // 2. Locale detection & redirect — check URL, cookie, Accept-Language header
  const pathnameLocale = SUPPORTED_LOCALES.find(
    (locale) => pathname.startsWith(`/${locale}/`) || pathname === `/${locale}`
  );

  if (!pathnameLocale) {
    const preferredLocale =
      request.cookies.get("locale")?.value ??
      negotiateLocale(request.headers.get("accept-language")) ??
      store.defaultLocale ??
      DEFAULT_LOCALE;

    return NextResponse.redirect(
      new URL(`/${preferredLocale}${pathname}`, request.url)
    );
  }

  // 3. Auth check for protected routes
  const pathWithoutLocale = pathname.replace(`/${pathnameLocale}`, "") || "/";
  if (!publicPaths.some((p) => pathWithoutLocale.startsWith(p)) && pathWithoutLocale !== "/") {
    const token = request.cookies.get("session-token")?.value;
    if (!token) {
      const loginUrl = new URL(`/${pathnameLocale}/login`, request.url);
      loginUrl.searchParams.set("callbackUrl", pathname);
      return NextResponse.redirect(loginUrl);
    }
  }

  return response;
}

function negotiateLocale(acceptLanguage: string | null): string | undefined {
  if (!acceptLanguage) return undefined;
  const preferred = acceptLanguage.split(",")[0]?.split("-")[0]?.trim();
  return SUPPORTED_LOCALES.find((l) => l === preferred);
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|api/).*)"],
};
```

### i18n Config — `src/i18n/config.ts`

```ts
export const SUPPORTED_LOCALES = ["en", "fr", "de", "ar", "ja"] as const;
export type Locale = (typeof SUPPORTED_LOCALES)[number];

export const DEFAULT_LOCALE: Locale = "en";

export const RTL_LOCALES: Locale[] = ["ar"];

export const LOCALE_LABELS: Record<Locale, string> = {
  en: "English",
  fr: "Français",
  de: "Deutsch",
  ar: "العربية",
  ja: "日本語",
};
```

### Currencies Config — `src/config/currencies.ts`

```ts
import type { Currency } from "@/types/currency";

export const SUPPORTED_CURRENCIES: Currency[] = ["USD", "EUR", "GBP", "JPY", "AED", "INR"];
export const DEFAULT_CURRENCY: Currency = "USD";

export const CURRENCY_CONFIG: Record<Currency, { symbol: string; decimals: number; locale: string }> = {
  USD: { symbol: "$", decimals: 2, locale: "en-US" },
  EUR: { symbol: "€", decimals: 2, locale: "de-DE" },
  GBP: { symbol: "£", decimals: 2, locale: "en-GB" },
  JPY: { symbol: "¥", decimals: 0, locale: "ja-JP" },
  AED: { symbol: "د.إ", decimals: 2, locale: "ar-AE" },
  INR: { symbol: "₹", decimals: 2, locale: "en-IN" },
};
```

### Store Config — `src/config/stores.ts`

> Each store maps to a region/domain with its own default locale, currency, and theme overrides.

```ts
import type { StoreConfig } from "@/types/store";
import type { Locale } from "@/i18n/config";

export const STORES: StoreConfig[] = [
  {
    id: "us",
    name: "United States",
    domain: "us.myapp.com",
    defaultLocale: "en",
    defaultCurrency: "USD",
    supportedCurrencies: ["USD"],
    supportedLocales: ["en"],
    theme: {},
  },
  {
    id: "eu",
    name: "Europe",
    domain: "eu.myapp.com",
    defaultLocale: "de",
    defaultCurrency: "EUR",
    supportedCurrencies: ["EUR", "GBP"],
    supportedLocales: ["en", "fr", "de"],
    theme: {},
  },
  {
    id: "me",
    name: "Middle East",
    domain: "me.myapp.com",
    defaultLocale: "ar",
    defaultCurrency: "AED",
    supportedCurrencies: ["AED", "USD"],
    supportedLocales: ["ar", "en"],
    theme: {},
  },
  {
    id: "in",
    name: "India",
    domain: "in.myapp.com",
    defaultLocale: "en",
    defaultCurrency: "INR",
    supportedCurrencies: ["INR", "USD"],
    supportedLocales: ["en"],
    theme: {},
  },
];

export function resolveStore(hostname: string): StoreConfig {
  return STORES.find((s) => s.domain === hostname) ?? STORES[0];
}

export function getStoreByLocale(locale: Locale): StoreConfig {
  return STORES.find((s) => s.defaultLocale === locale) ?? STORES[0];
}
```

### Next.js Config — `next.config.ts`

```ts
import type { NextConfig } from "next";
import createNextIntlPlugin from "next-intl/plugin";

const withNextIntl = createNextIntlPlugin("./src/i18n/config.ts");

const nextConfig: NextConfig = {
  output: "standalone",
  typedRoutes: true,
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "*.amazonaws.com" },
    ],
  },
  experimental: {
    authInterrupts: true,
  },
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          { key: "X-Frame-Options", value: "DENY" },
          { key: "X-Content-Type-Options", value: "nosniff" },
          { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
        ],
      },
    ];
  },
};

export default withNextIntl(nextConfig);
```

### Metadata Helper — `src/lib/metadata.ts`

> Generates locale-aware metadata with hreflang alternates for SEO. Used by every page via `generateMetadata()`.

```ts
import type { Metadata } from "next";
import { SUPPORTED_LOCALES } from "@/i18n/config";

interface PageMetadataOptions {
  locale: string;
  titleKey: string;
  descKey: string;
  path?: string;
  images?: string[];
}

export function generatePageMetadata({ locale, titleKey, descKey, path = "", images }: PageMetadataOptions): Metadata {
  const baseUrl = process.env.NEXT_PUBLIC_APP_URL!;

  return {
    title: titleKey,
    description: descKey,
    alternates: {
      canonical: `${baseUrl}/${locale}${path}`,
      languages: Object.fromEntries(
        SUPPORTED_LOCALES.map((l) => [l, `${baseUrl}/${l}${path}`])
      ),
    },
    openGraph: {
      locale,
      alternateLocale: SUPPORTED_LOCALES.filter((l) => l !== locale),
      images: images ?? [`${baseUrl}/og-image.png`],
    },
  };
}
```

### Web Vitals Reporting — `src/lib/web-vitals.ts`

> Reports Core Web Vitals (CLS, LCP, INP, FCP, TTFB) to your analytics provider.

```ts
import type { Metric } from "web-vitals";

const vitalsUrl = "/api/analytics/vitals";

export function reportWebVitals(metric: Metric) {
  const body = {
    id: metric.id,
    name: metric.name,
    value: metric.value,
    rating: metric.rating,         // "good" | "needs-improvement" | "poor"
    delta: metric.delta,
    navigationType: metric.navigationType,
    page: window.location.pathname, // e.g., /en/products
  };

  // Use sendBeacon for reliability during page unload
  if (navigator.sendBeacon) {
    navigator.sendBeacon(vitalsUrl, JSON.stringify(body));
  }
}
```

### TypeScript Config — `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "esnext",
    "lib": ["dom", "dom.iterable", "esnext"],
    "strict": true,
    "noEmit": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### Environment Variables — `.env.local`

```env
# .env.local — NEVER COMMIT THIS FILE
AUTH_SECRET="your-secret-here"
AUTH_URL="http://localhost:3000"

# Third-party
STRIPE_SECRET_KEY="sk_test_..."
STRIPE_WEBHOOK_SECRET="whsec_..."
RESEND_API_KEY="re_..."

# ─── Backend Provider ───────────────────────────────────────────────────────
# Set ONE provider. Only configure the env vars for the active provider.
# Options: magento | shopify | odoo | custom
BACKEND_PROVIDER="magento"

# Magento 2 (GraphQL)
MAGENTO_GRAPHQL_URL="https://your-magento.com/graphql"
MAGENTO_ADMIN_TOKEN="your-magento-admin-token"

# Shopify Storefront API (GraphQL)
SHOPIFY_STOREFRONT_URL="https://your-store.myshopify.com/api/2024-01/graphql.json"
SHOPIFY_STOREFRONT_TOKEN="your-storefront-api-token"

# Odoo (REST)
ODOO_BASE_URL="https://your-odoo.com"
ODOO_DB="your-database-name"
ODOO_API_KEY="your-odoo-api-key"

# Custom / any other backend
CUSTOM_API_URL="https://your-api.com"
CUSTOM_API_KEY="your-api-key"
# ─────────────────────────────────────────────────────────────────────────────

# Multi-store
NEXT_PUBLIC_DEFAULT_STORE="us"
EXCHANGE_RATE_API_KEY="your-exchange-rate-api-key"
EXCHANGE_RATE_API_URL="https://api.exchangerate.host/latest"

# i18n
NEXT_PUBLIC_DEFAULT_LOCALE="en"

# Public (exposed to browser — baked in at build time)
NEXT_PUBLIC_APP_URL="http://localhost:3000"
NEXT_PUBLIC_POSTHOG_KEY="phc_..."
```

### Dockerfile — Multi-Stage Build

```dockerfile
FROM node:22-alpine AS base

FROM base AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable pnpm && pnpm install --frozen-lockfile

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN corepack enable pnpm && pnpm build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

### CI/CD Pipeline — `.github/workflows/ci.yml`

```yaml
name: CI
on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: pnpm/action-setup@v6
        with: { version: 9 }
      - uses: actions/setup-node@v6
        with: { node-version: 22, cache: "pnpm" }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm type-check
      - run: pnpm test
      - run: pnpm build
```

### Unit Test: Utility Functions — `src/lib/utils.test.ts`

```ts
import { describe, it, expect } from "vitest";
import { cn, formatDate } from "@/lib/utils";
import { formatPrice } from "@/lib/currency";

describe("cn", () => {
  it("merges class names", () => {
    expect(cn("px-4", "py-2")).toBe("px-4 py-2");
  });

  it("handles conditional classes", () => {
    expect(cn("base", false && "hidden", "visible")).toBe("base visible");
  });

  it("deduplicates tailwind conflicts", () => {
    expect(cn("px-4", "px-8")).toBe("px-8");
  });
});

describe("formatPrice", () => {
  it("formats USD by default", () => {
    expect(formatPrice(29.99)).toBe("$29.99");
  });

  it("formats EUR with locale", () => {
    expect(formatPrice(29.99, "EUR", "de-DE")).toContain("29,99");
  });

  it("formats JPY with zero decimals", () => {
    expect(formatPrice(2999, "JPY", "ja-JP")).toContain("2,999");
  });

  it("formats large numbers with commas", () => {
    expect(formatPrice(1299.99)).toBe("$1,299.99");
  });
});

describe("formatDate", () => {
  it("formats a date", () => {
    const date = new Date("2026-01-15");
    expect(formatDate(date)).toBe("Jan 15, 2026");
  });
});
```

### Unit Test: Product Card — `src/components/product/product-card.test.tsx`

> **Vitest + Testing Library:** Test component rendering with mocked GraphQL data.

```tsx
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import { ProductCard } from "@/components/product/product-card";

const mockProduct = {
  documentId: "1",
  name: "Wireless Headphones",
  slug: "wireless-headphones",
  price: 79.99,
  compareAtPrice: 99.99,
  inStock: true,
  description: "Premium wireless headphones",
  category: { documentId: "cat-1", name: "Audio", slug: "audio" },
  images: [
    { url: "/images/headphones.jpg", alternativeText: "Headphones", width: 800, height: 800 },
  ],
  createdAt: "2026-01-01",
};

describe("ProductCard", () => {
  it("renders product name and price", () => {
    render(<ProductCard product={mockProduct} />);
    expect(screen.getByText("Wireless Headphones")).toBeInTheDocument();
    expect(screen.getByText("$79.99")).toBeInTheDocument();
  });

  it("shows sale badge when compareAtPrice exists", () => {
    render(<ProductCard product={mockProduct} />);
    expect(screen.getByText("Sale")).toBeInTheDocument();
    expect(screen.getByText("$99.99")).toBeInTheDocument();
  });

  it("shows out of stock badge when not in stock", () => {
    render(<ProductCard product={{ ...mockProduct, inStock: false }} />);
    expect(screen.getByText("Out of stock")).toBeInTheDocument();
  });

  it("renders category name", () => {
    render(<ProductCard product={mockProduct} />);
    expect(screen.getByText("Audio")).toBeInTheDocument();
  });

  it("links to product detail page", () => {
    render(<ProductCard product={mockProduct} />);
    const link = screen.getByRole("link");
    expect(link).toHaveAttribute("href", "/products/wireless-headphones");
  });
});
```

### Unit Test: Cart Store — `src/stores/cart.store.test.ts`

```ts
import { describe, it, expect, beforeEach } from "vitest";
import { useCartStore } from "@/stores/cart.store";

describe("useCartStore", () => {
  beforeEach(() => {
    useCartStore.setState({ cartId: "" });
  });

  it("has empty cartId by default", () => {
    expect(useCartStore.getState().cartId).toBe("");
  });

  it("sets cartId", () => {
    useCartStore.getState().setCartId("cart-123");
    expect(useCartStore.getState().cartId).toBe("cart-123");
  });
});
```

### Unit Test: useDebounce Hook — `src/hooks/use-debounce.test.ts`

```ts
import { describe, it, expect, vi } from "vitest";
import { renderHook, act } from "@testing-library/react";
import { useDebounce } from "@/hooks/use-debounce";

describe("useDebounce", () => {
  it("returns initial value immediately", () => {
    const { result } = renderHook(() => useDebounce("hello", 500));
    expect(result.current).toBe("hello");
  });

  it("debounces value changes", async () => {
    vi.useFakeTimers();
    const { result, rerender } = renderHook(
      ({ value, delay }) => useDebounce(value, delay),
      { initialProps: { value: "hello", delay: 500 } }
    );

    rerender({ value: "world", delay: 500 });
    expect(result.current).toBe("hello");

    act(() => {
      vi.advanceTimersByTime(500);
    });
    expect(result.current).toBe("world");

    vi.useRealTimers();
  });
});
```

### Unit Test: Add to Cart Button — `src/components/product/add-to-cart-button.test.tsx`

```tsx
import { describe, it, expect, vi } from "vitest";
import { render, screen, fireEvent } from "@testing-library/react";
import { AddToCartButton } from "@/components/product/add-to-cart-button";

// Mock the cart server action — backend-agnostic, no Apollo needed
vi.mock("@/actions/cart.actions", () => ({
  addToCartAction: vi.fn().mockResolvedValue({
    id: "cart-1",
    itemCount: 1,
    total: 79.99,
  }),
}));

vi.mock("@/stores/cart.store", () => ({
  useCartStore: (selector: (s: { cartId: string }) => string) =>
    selector({ cartId: "cart-1" }),
}));

describe("AddToCartButton", () => {
  it("renders with default quantity of 1", () => {
    render(<AddToCartButton productId="prod-1" />);
    expect(screen.getByText("1")).toBeInTheDocument();
    expect(screen.getByText("Add to Cart")).toBeInTheDocument();
  });

  it("increments and decrements quantity", () => {
    render(<AddToCartButton productId="prod-1" />);
    fireEvent.click(screen.getByText("+"));
    expect(screen.getByText("2")).toBeInTheDocument();

    fireEvent.click(screen.getByText("-"));
    expect(screen.getByText("1")).toBeInTheDocument();
  });

  it("does not go below quantity 1", () => {
    render(<AddToCartButton productId="prod-1" />);
    fireEvent.click(screen.getByText("-"));
    expect(screen.getByText("1")).toBeInTheDocument();
  });

  it("shows Out of Stock when disabled", () => {
    render(<AddToCartButton productId="prod-1" disabled />);
    expect(screen.getByText("Out of Stock")).toBeInTheDocument();
    expect(screen.getByRole("button", { name: "Out of Stock" })).toBeDisabled();
  });
});
```

### Vitest Config — `vitest.config.ts`

> **Co-located tests:** Unit tests live next to source files (`*.test.ts` / `*.test.tsx`). Vitest scans `src/` for them automatically. E2E tests stay in `tests/e2e/`.

```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./tests/setup.ts"],
    include: ["src/**/*.test.{ts,tsx}"],
    exclude: ["src/data-layer/adapters/**/generated/**"],
    coverage: {
      provider: "v8",
      include: ["src/**/*.{ts,tsx}"],
      exclude: ["src/**/*.test.{ts,tsx}", "src/data-layer/adapters/**/generated/**"],
    },
  },
});
```

### Test Setup — `tests/setup.ts`

```ts
import "@testing-library/jest-dom/vitest";
```

### E2E Test: Auth — `tests/e2e/auth.spec.ts`

> **Playwright:** Use locator-based API (`getByRole`, `getByLabel`) instead of CSS selectors for resilient tests.

```ts
import { test, expect } from "@playwright/test";

test("user can log in", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill("test@example.com");
  await page.getByLabel("Password").fill("password123");
  await page.getByRole("button", { name: "Sign in" }).click();
  await expect(page).toHaveURL("/overview");
  await expect(page.getByRole("heading", { level: 1 })).toContainText("Dashboard");
});
```

### E2E Test: Products — `tests/e2e/products.spec.ts`

```ts
import { test, expect } from "@playwright/test";

test.describe("Product Listing", () => {
  test("displays products on the listing page", async ({ page }) => {
    await page.goto("/products");
    await expect(page.getByRole("heading", { name: "Products" })).toBeVisible();
    const productCards = page.locator('[href^="/products/"]');
    await expect(productCards.first()).toBeVisible();
  });

  test("filters products by category", async ({ page }) => {
    await page.goto("/products");
    await page.getByText("Audio").click();
    await expect(page).toHaveURL(/category=audio/);
  });

  test("sorts products by price", async ({ page }) => {
    await page.goto("/products");
    await page.getByText("Price: Low to High").click();
    await expect(page).toHaveURL(/sort=price%3Aasc/);
  });
});

test.describe("Product Detail", () => {
  test("shows product info on detail page", async ({ page }) => {
    await page.goto("/products");
    const firstProduct = page.locator('[href^="/products/"]').first();
    const productName = await firstProduct.locator("h3").textContent();
    await firstProduct.click();
    await expect(page.getByRole("heading", { level: 1 })).toContainText(productName!);
  });

  test("can add product to cart", async ({ page }) => {
    await page.goto("/products");
    await page.locator('[href^="/products/"]').first().click();
    await page.getByRole("button", { name: "Add to Cart" }).click();
    await expect(page.getByRole("button", { name: "Add to Cart" })).toBeEnabled();
  });
});
```

### E2E Test: Cart — `tests/e2e/cart.spec.ts`

```ts
import { test, expect } from "@playwright/test";

test.describe("Shopping Cart", () => {
  test("can add item and view cart", async ({ page }) => {
    await page.goto("/products");
    await page.locator('[href^="/products/"]').first().click();
    await page.getByRole("button", { name: "Add to Cart" }).click();
    await page.goto("/cart");
    await expect(page.locator("body")).toContainText("Subtotal");
  });

  test("can update quantity in cart", async ({ page }) => {
    await page.goto("/cart");
    const incrementBtn = page.getByText("+").first();
    if (await incrementBtn.isVisible()) {
      await incrementBtn.click();
    }
  });

  test("can proceed to checkout", async ({ page }) => {
    await page.goto("/cart");
    const checkoutBtn = page.getByRole("link", { name: /checkout/i });
    if (await checkoutBtn.isVisible()) {
      await checkoutBtn.click();
      await expect(page).toHaveURL("/checkout");
    }
  });
});
```

### Test Fixture: Products — `tests/fixtures/products.ts`

```ts
export function createMockProduct(overrides = {}) {
  return {
    documentId: "prod-1",
    name: "Wireless Headphones",
    slug: "wireless-headphones",
    description: "Premium wireless headphones with noise cancellation",
    price: 79.99,
    compareAtPrice: 99.99,
    inStock: true,
    sku: "WH-001",
    category: {
      documentId: "cat-1",
      name: "Audio",
      slug: "audio",
    },
    images: [
      {
        url: "/images/headphones.jpg",
        alternativeText: "Wireless Headphones",
        width: 800,
        height: 800,
      },
    ],
    variants: [],
    relatedProducts: [],
    createdAt: "2026-01-01T00:00:00.000Z",
    ...overrides,
  };
}

export function createMockProductList(count = 6) {
  return Array.from({ length: count }, (_, i) =>
    createMockProduct({
      documentId: `prod-${i + 1}`,
      name: `Product ${i + 1}`,
      slug: `product-${i + 1}`,
      price: 19.99 + i * 10,
    })
  );
}
```

### Auth.js v5 Config — `src/lib/auth.ts`

> **Auth.js v5:** Single `auth()` function replaces `getServerSession()`. Works in Server Components, Route Handlers, Proxy, and Server Actions.

```ts
import NextAuth from "next-auth";
import GitHub from "next-auth/providers/github";
import Credentials from "next-auth/providers/credentials";

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    GitHub,
    Credentials({
      credentials: {
        email: { label: "Email" },
        password: { label: "Password", type: "password" },
      },
      authorize: async (credentials) => {
        // Validate credentials against your GraphQL API
        return null;
      },
    }),
  ],
  pages: {
    signIn: "/login",
  },
});
```

### Auth.js Route Handler — `src/app/api/auth/[...nextauth]/route.ts`

```ts
import { handlers } from "@/lib/auth";

export const { GET, POST } = handlers;
```

### Server Action — `src/actions/user.actions.ts`

> **React 19:** Use Server Actions with `useActionState` for form handling. Replaces the deprecated `useFormState`.

```ts
"use server";

import { revalidatePath } from "next/cache";
import { getBackend } from "@/data-layer/factory";
import { createUserSchema } from "@/lib/validations/user";

export async function createUser(prevState: unknown, formData: FormData) {
  const parsed = createUserSchema.safeParse({
    name: formData.get("name"),
    email: formData.get("email"),
    role: formData.get("role"),
  });

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors };
  }

  const { users } = getBackend();
  await users.createUser(parsed.data);

  revalidatePath("/users");
  return { success: true };
}
```

### Form with useActionState — `src/components/forms/user-form.tsx`

> **React 19 hooks:** `useActionState` tracks pending/error/success. `useFormStatus` provides submission status without prop drilling. `useOptimistic` shows instant UI feedback.

```tsx
"use client";

import { useActionState } from "react";
import { useFormStatus } from "react-dom";
import { createUser } from "@/actions/user.actions";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <Button type="submit" disabled={pending}>
      {pending ? "Creating..." : "Create User"}
    </Button>
  );
}

export function UserForm() {
  const [state, formAction, isPending] = useActionState(createUser, null);

  return (
    <form action={formAction}>
      <Input name="name" placeholder="Name" required />
      <Input name="email" type="email" placeholder="Email" required />
      {state?.error && (
        <p className="text-sm text-destructive">{JSON.stringify(state.error)}</p>
      )}
      <SubmitButton />
    </form>
  );
}
```

### Async Params in Pages — `src/app/(dashboard)/users/[id]/page.tsx`

> **Next.js 16:** `params` and `searchParams` are async Promises and must be awaited (introduced in Next.js 15, fully enforced in 16).

```tsx
import { getUserById } from "@/services/user.service";
import { notFound } from "next/navigation";

export default async function UserPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const user = await getUserById(id);
  if (!user) notFound();

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### Backend Adapter Pattern

> The entire data layer is backend-agnostic. Define canonical domain types and repository interfaces once. Implement them per backend. Switch backends by changing a single env var — zero changes to services, components, or pages.

---

#### Canonical Domain Types — `src/data-layer/types/product.ts`

> These are YOUR types — not Magento types, not Shopify types. Every adapter maps its backend response into these shapes.

```ts
export interface Product {
  id: string;
  slug: string;
  name: string;
  description: string;
  price: number;
  compareAtPrice?: number;
  currency: string;
  images: ProductImage[];
  categories: string[];
  inStock: boolean;
  variants?: ProductVariant[];
}

export interface ProductImage {
  url: string;
  alt: string;
  width: number;
  height: number;
}

export interface ProductVariant {
  id: string;
  sku: string;
  title: string;
  price: number;
  inStock: boolean;
  attributes: Record<string, string>; // e.g. { color: "red", size: "M" }
}

export interface ProductFilters {
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  inStock?: boolean;
  search?: string;
  page?: number;
  limit?: number;
}

export interface PaginatedResult<T> {
  items: T[];
  total: number;
  page: number;
  limit: number;
  hasNextPage: boolean;
}
```

#### Canonical Domain Types — `src/data-layer/types/cart.ts`

```ts
export interface Cart {
  id: string;
  items: CartItem[];
  subtotal: number;
  tax: number;
  total: number;
  currency: string;
  itemCount: number;
}

export interface CartItem {
  id: string;
  productId: string;
  variantId?: string;
  name: string;
  image: string;
  price: number;
  quantity: number;
  slug: string;
}
```

---

#### Repository Interface — `src/data-layer/interfaces/product.repository.ts`

> The contract. Every adapter must implement this interface exactly. If a backend can't support a method, it throws `NotImplementedError`.

```ts
import type { Product, ProductFilters, PaginatedResult } from "@/data-layer/types/product";

export interface IProductRepository {
  getProducts(filters?: ProductFilters): Promise<PaginatedResult<Product>>;
  getProductBySlug(slug: string): Promise<Product | null>;
  getProductCategories(): Promise<Category[]>;
}
```

#### Repository Interface — `src/data-layer/interfaces/cart.repository.ts`

```ts
import type { Cart } from "@/data-layer/types/cart";

export interface ICartRepository {
  getCart(cartId: string): Promise<Cart | null>;
  createCart(): Promise<Cart>;
  addItem(cartId: string, productId: string, quantity: number, variantId?: string): Promise<Cart>;
  updateItem(cartId: string, itemId: string, quantity: number): Promise<Cart>;
  removeItem(cartId: string, itemId: string): Promise<Cart>;
  clearCart(cartId: string): Promise<void>;
}
```

#### Repository Interface — `src/data-layer/interfaces/user.repository.ts`

```ts
import type { User } from "@/data-layer/types/user";

export interface IUserRepository {
  getUsers(opts?: { page?: number; limit?: number }): Promise<PaginatedResult<User>>;
  getUserById(id: string): Promise<User | null>;
  createUser(input: CreateUserInput): Promise<User>;
  updateUser(id: string, input: UpdateUserInput): Promise<User>;
  deleteUser(id: string): Promise<void>;
}
```

---

#### Backend Factory — `src/data-layer/factory.ts`

> Reads `BACKEND_PROVIDER` at startup and returns the correct adapter. Called once — result is cached in module scope.

```ts
import type { IProductRepository } from "./interfaces/product.repository";
import type { ICartRepository } from "./interfaces/cart.repository";
import type { IUserRepository } from "./interfaces/user.repository";

export interface BackendAdapter {
  products: IProductRepository;
  cart: ICartRepository;
  users: IUserRepository;
}

let _backend: BackendAdapter | null = null;

export function getBackend(): BackendAdapter {
  if (_backend) return _backend;

  const provider = process.env.BACKEND_PROVIDER;

  switch (provider) {
    case "magento":
      _backend = require("./adapters/magento").MagentoAdapter;
      break;
    case "shopify":
      _backend = require("./adapters/shopify").ShopifyAdapter;
      break;
    case "odoo":
      _backend = require("./adapters/odoo").OdooAdapter;
      break;
    case "custom":
    default:
      _backend = require("./adapters/custom").CustomAdapter;
      break;
  }

  if (!_backend) {
    throw new Error(
      `Unknown BACKEND_PROVIDER: "${provider}". Valid options: magento | shopify | odoo | custom`
    );
  }

  return _backend;
}
```

---

#### Magento Adapter — `src/data-layer/adapters/magento/client.ts`

> Uses `graphql-request` (5KB) instead of Apollo (~40KB). Custom `fetch` override wires into Next.js's native cache for free ISR/tags/revalidation.

```ts
import { GraphQLClient } from "graphql-request";

if (!process.env.MAGENTO_GRAPHQL_URL) {
  throw new Error("MAGENTO_GRAPHQL_URL is not set");
}

export const magentoClient = new GraphQLClient(process.env.MAGENTO_GRAPHQL_URL, {
  headers: { Authorization: `Bearer ${process.env.MAGENTO_ADMIN_TOKEN}` },
  fetch: (url, init) =>
    fetch(url, { ...init, next: { revalidate: 60, tags: ["magento"] } }),
});
```

#### Magento Adapter — `src/data-layer/adapters/magento/product.adapter.ts`

> Queries are inline via `gql.tada` — no codegen step, no `generated/` folder. Types are inferred at edit time via the `@0no-co/graphqlsp` LSP plugin.

```ts
import { graphql } from "gql.tada";
import { magentoClient } from "./client";
import type { IProductRepository } from "@/data-layer/interfaces/product.repository";
import type { Product, ProductFilters, PaginatedResult } from "@/data-layer/types/product";

const ProductsQuery = graphql(`
  query Products($pageSize: Int!, $currentPage: Int!, $filter: ProductAttributeFilterInput) {
    products(pageSize: $pageSize, currentPage: $currentPage, filter: $filter) {
      total_count
      page_info { current_page page_size }
      items {
        uid
        name
        url_key
        description { html }
        stock_status
        price_range { minimum_price { regular_price { value currency } } }
        media_gallery { url label }
        categories { name }
      }
    }
  }
`);

const ProductByUrlKeyQuery = graphql(`
  query ProductByUrlKey($urlKey: String!) {
    products(filter: { url_key: { eq: $urlKey } }) {
      items {
        uid name url_key description { html } stock_status
        price_range { minimum_price { regular_price { value currency } } }
        media_gallery { url label }
        categories { name }
      }
    }
  }
`);

export class MagentoProductAdapter implements IProductRepository {
  async getProducts(filters?: ProductFilters): Promise<PaginatedResult<Product>> {
    const data = await magentoClient.request(ProductsQuery, {
      pageSize: filters?.limit ?? 20,
      currentPage: filters?.page ?? 1,
      filter: filters?.category ? { category_uid: { eq: filters.category } } : {},
    });

    return {
      items: data.products.items.map(normalizeMagentoProduct),
      total: data.products.total_count,
      page: filters?.page ?? 1,
      limit: filters?.limit ?? 20,
      hasNextPage: (filters?.page ?? 1) * (filters?.limit ?? 20) < data.products.total_count,
    };
  }

  async getProductBySlug(slug: string): Promise<Product | null> {
    const data = await magentoClient.request(ProductByUrlKeyQuery, { urlKey: slug });
    const item = data.products.items[0];
    return item ? normalizeMagentoProduct(item) : null;
  }

  async getProductCategories() {
    return []; // implement with CategoryListQuery
  }
}

// Normalization — keeps Magento field names out of your app
function normalizeMagentoProduct(item: any): Product {
  return {
    id: item.uid,
    slug: item.url_key,                         // Magento uses url_key, not slug
    name: item.name,
    description: item.description?.html ?? "",
    price: item.price_range.minimum_price.regular_price.value,
    currency: item.price_range.minimum_price.regular_price.currency,
    inStock: item.stock_status === "IN_STOCK",  // Magento uses stock_status enum
    images: item.media_gallery.map((img: any) => ({
      url: img.url,
      alt: img.label ?? item.name,
      width: 800,
      height: 800,
    })),
    categories: item.categories?.map((c: any) => c.name) ?? [],
    variants: [],
  };
}
```

---

#### Shopify Adapter — `src/data-layer/adapters/shopify/client.ts`

```ts
import { GraphQLClient } from "graphql-request";

export const shopifyClient = new GraphQLClient(process.env.SHOPIFY_STOREFRONT_URL!, {
  headers: {
    "X-Shopify-Storefront-Access-Token": process.env.SHOPIFY_STOREFRONT_TOKEN!,
    "Content-Type": "application/json",
  },
  fetch: (url, init) =>
    fetch(url, { ...init, next: { revalidate: 60, tags: ["shopify"] } }),
});
```

#### Shopify Adapter — `src/data-layer/adapters/shopify/product.adapter.ts`

```ts
import { graphql } from "gql.tada";
import { shopifyClient } from "./client";
import type { IProductRepository } from "@/data-layer/interfaces/product.repository";
import type { Product, ProductFilters, PaginatedResult } from "@/data-layer/types/product";

const ProductsQuery = graphql(`
  query Products($first: Int!) {
    products(first: $first) {
      pageInfo { hasNextPage }
      edges {
        node {
          id handle title description availableForSale
          priceRange { minVariantPrice { amount currencyCode } }
          images(first: 10) { edges { node { url altText width height } } }
          collections(first: 5) { edges { node { title } } }
          variants(first: 50) {
            edges {
              node {
                id sku title availableForSale
                price { amount }
                selectedOptions { name value }
              }
            }
          }
        }
      }
    }
  }
`);

const ProductByHandleQuery = graphql(`
  query ProductByHandle($handle: String!) {
    product(handle: $handle) {
      id handle title description availableForSale
      priceRange { minVariantPrice { amount currencyCode } }
      images(first: 10) { edges { node { url altText width height } } }
      collections(first: 5) { edges { node { title } } }
    }
  }
`);

export class ShopifyProductAdapter implements IProductRepository {
  async getProducts(filters?: ProductFilters): Promise<PaginatedResult<Product>> {
    const data = await shopifyClient.request(ProductsQuery, { first: filters?.limit ?? 20 });

    return {
      items: data.products.edges.map((e) => normalizeShopifyProduct(e.node)),
      total: data.products.edges.length,
      page: 1,
      limit: filters?.limit ?? 20,
      hasNextPage: data.products.pageInfo.hasNextPage,
    };
  }

  async getProductBySlug(slug: string): Promise<Product | null> {
    // Shopify uses handle, not slug
    const data = await shopifyClient.request(ProductByHandleQuery, { handle: slug });
    return data.product ? normalizeShopifyProduct(data.product) : null;
  }

  async getProductCategories() {
    return []; // Use Shopify Collections as categories
  }
}

function normalizeShopifyProduct(node: any): Product {
  return {
    id: node.id,
    slug: node.handle,                          // Shopify uses handle, not slug
    name: node.title,
    description: node.description,
    price: parseFloat(node.priceRange.minVariantPrice.amount),
    currency: node.priceRange.minVariantPrice.currencyCode,
    inStock: node.availableForSale,             // Shopify uses availableForSale
    images: node.images.edges.map((e: any) => ({
      url: e.node.url,
      alt: e.node.altText ?? node.title,
      width: e.node.width ?? 800,
      height: e.node.height ?? 800,
    })),
    categories: node.collections?.edges.map((e: any) => e.node.title) ?? [],
    variants: node.variants.edges.map((e: any) => ({
      id: e.node.id,
      sku: e.node.sku,
      title: e.node.title,
      price: parseFloat(e.node.price.amount),
      inStock: e.node.availableForSale,
      attributes: Object.fromEntries(
        e.node.selectedOptions.map((o: any) => [o.name.toLowerCase(), o.value])
      ),
    })),
  };
}
```

---

#### Odoo Adapter — `src/data-layer/adapters/odoo/client.ts`

```ts
const ODOO_BASE = process.env.ODOO_BASE_URL!;
const ODOO_DB   = process.env.ODOO_DB!;
const ODOO_KEY  = process.env.ODOO_API_KEY!;

export async function odooFetch<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  const res = await fetch(`${ODOO_BASE}${endpoint}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      "api-key": ODOO_KEY,
      "db": ODOO_DB,
      ...options.headers,
    },
  });

  if (!res.ok) {
    throw new Error(`Odoo API error: ${res.status} ${res.statusText}`);
  }

  return res.json() as T;
}
```

#### Odoo Adapter — `src/data-layer/adapters/odoo/product.adapter.ts`

```ts
import { odooFetch } from "./client";
import type { IProductRepository } from "@/data-layer/interfaces/product.repository";
import type { Product, ProductFilters, PaginatedResult } from "@/data-layer/types/product";

export class OdooProductAdapter implements IProductRepository {
  async getProducts(filters?: ProductFilters): Promise<PaginatedResult<Product>> {
    const params = new URLSearchParams({
      page: String(filters?.page ?? 1),
      page_size: String(filters?.limit ?? 20),
      ...(filters?.category && { category: filters.category }),
      ...(filters?.search && { name: filters.search }),
    });

    const data = await odooFetch<OdooProductListResponse>(`/api/v1/products?${params}`);

    return {
      items: data.results.map(normalizeOdooProduct),
      total: data.count,
      page: filters?.page ?? 1,
      limit: filters?.limit ?? 20,
      hasNextPage: !!data.next,
    };
  }

  async getProductBySlug(slug: string): Promise<Product | null> {
    try {
      const data = await odooFetch<OdooProduct>(`/api/v1/products/${slug}`);
      return normalizeOdooProduct(data);
    } catch {
      return null;
    }
  }

  async getProductCategories() {
    const data = await odooFetch<{ results: any[] }>("/api/v1/product-categories");
    return data.results.map((c) => ({ id: c.id, name: c.name, slug: c.slug }));
  }
}

function normalizeOdooProduct(item: any): Product {
  return {
    id: String(item.id),
    slug: item.website_slug ?? String(item.id),
    name: item.name,
    description: item.description_sale ?? "",
    price: item.list_price,
    currency: item.currency_id?.[1] ?? "USD",
    inStock: item.qty_available > 0,
    images: item.image_1920
      ? [{ url: `${process.env.ODOO_BASE_URL}/web/image/product.product/${item.id}/image_1920`, alt: item.name, width: 800, height: 800 }]
      : [],
    categories: item.categ_id ? [item.categ_id[1]] : [],
    variants: [],
  };
}
```

---

#### Exporting the Adapter — `src/data-layer/adapters/magento/index.ts`

```ts
import { MagentoProductAdapter } from "./product.adapter";
import { MagentoCartAdapter } from "./cart.adapter";
import { MagentoUserAdapter } from "./user.adapter";
import type { BackendAdapter } from "@/data-layer/factory";

export const MagentoAdapter: BackendAdapter = {
  products: new MagentoProductAdapter(),
  cart: new MagentoCartAdapter(),
  users: new MagentoUserAdapter(),
};
```

---

#### Using the Adapter in a Server Component

```tsx
// src/app/(shop)/products/page.tsx
import { getProducts } from "@/services/product.service";

export default async function ProductsPage({ searchParams }: { searchParams: Promise<{ category?: string }> }) {
  const { category } = await searchParams;
  const { items, total } = await getProducts({ category });  // backend-agnostic

  return (
    <main>
      <p>{total} products</p>
      <ProductGrid products={items} />
    </main>
  );
}
```

#### Using the Adapter in a Server Action

```ts
// src/actions/cart.actions.ts
"use server";

import { getBackend } from "@/data-layer/factory";
import { revalidatePath } from "next/cache";

export async function addToCartAction(cartId: string, productId: string, quantity: number) {
  const { cart } = getBackend();
  const updatedCart = await cart.addItem(cartId, productId, quantity);
  revalidatePath("/cart");
  return updatedCart;
}
```

#### Adding a New Backend (Custom Template) — `src/data-layer/adapters/custom/product.adapter.ts`

```ts
import type { IProductRepository } from "@/data-layer/interfaces/product.repository";
import type { Product, ProductFilters, PaginatedResult } from "@/data-layer/types/product";

export class CustomProductAdapter implements IProductRepository {
  async getProducts(filters?: ProductFilters): Promise<PaginatedResult<Product>> {
    // TODO: implement with your API
    throw new Error("Not implemented");
  }

  async getProductBySlug(slug: string): Promise<Product | null> {
    // TODO: implement with your API
    throw new Error("Not implemented");
  }

  async getProductCategories() {
    // TODO: implement with your API
    throw new Error("Not implemented");
  }
}
```

---

## Ecommerce Examples

> All ecommerce examples use the Backend Adapter Pattern. Pages and components import canonical types from `src/data-layer/types/` — never from backend-specific generated files. The active backend is resolved transparently via `getBackend()`.

### Product Service — `src/services/product.service.ts`

```ts
import { getBackend } from "@/data-layer/factory";
import { cache } from "react";
import type { ProductFilters } from "@/data-layer/types/product";

export const getProducts = cache(async (filters?: ProductFilters) => {
  const { products } = getBackend();
  return products.getProducts(filters);
});

export const getProductBySlug = cache(async (slug: string) => {
  const { products } = getBackend();
  return products.getProductBySlug(slug);
});

export const getProductCategories = cache(async () => {
  const { products } = getBackend();
  return products.getProductCategories();
});
```

### Cart Service — `src/services/cart.service.ts`

```ts
import { getBackend } from "@/data-layer/factory";
import { cache } from "react";

export const getCart = cache(async (cartId: string) => {
  const { cart } = getBackend();
  return cart.getCart(cartId);
});
```

### Product List Page — `src/app/(shop)/products/page.tsx`

> Server Component — fetches products via the backend-agnostic service. Works with Magento, Shopify, Odoo, or any adapter.

```tsx
import { getProducts } from "@/services/product.service";
import { ProductGrid } from "@/components/product/product-grid";
import { ProductFilters } from "@/components/product/product-filters";
import { generatePageMetadata } from "@/lib/metadata";

// ISR: revalidate product listing every 60 seconds
export const revalidate = 60;

export async function generateMetadata({ params }: { params: Promise<{ locale: string }> }) {
  const { locale } = await params;
  return generatePageMetadata({ locale, titleKey: "products.title", descKey: "products.description" });
}

interface ProductsPageProps {
  searchParams: Promise<{
    category?: string;
    minPrice?: string;
    maxPrice?: string;
    page?: string;
  }>;
}

export default async function ProductsPage({ searchParams }: ProductsPageProps) {
  const { category, minPrice, maxPrice, page } = await searchParams;

  const { items, total } = await getProducts({
    category,
    minPrice: minPrice ? parseFloat(minPrice) : undefined,
    maxPrice: maxPrice ? parseFloat(maxPrice) : undefined,
    page: page ? parseInt(page) : 1,
    limit: 12,
  });

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Products ({total})</h1>
      <div className="flex gap-8">
        <aside className="w-64 shrink-0">
          <ProductFilters activeCategory={category} minPrice={minPrice} maxPrice={maxPrice} />
        </aside>
        <main className="flex-1">
          <ProductGrid products={items} />
        </main>
      </div>
    </div>
  );
}
```

### Product List Loading — `src/app/(shop)/products/loading.tsx`

```tsx
export default function ProductsLoading() {
  return (
    <div className="container mx-auto px-4 py-8">
      <div className="h-9 w-48 bg-muted animate-pulse rounded mb-8" />
      <div className="flex gap-8">
        <aside className="w-64 shrink-0">
          <div className="space-y-4">
            {Array.from({ length: 5 }).map((_, i) => (
              <div key={i} className="h-6 bg-muted animate-pulse rounded" />
            ))}
          </div>
        </aside>
        <main className="flex-1 grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
          {Array.from({ length: 12 }).map((_, i) => (
            <div key={i} className="space-y-3">
              <div className="aspect-square bg-muted animate-pulse rounded-lg" />
              <div className="h-5 w-3/4 bg-muted animate-pulse rounded" />
              <div className="h-4 w-1/4 bg-muted animate-pulse rounded" />
            </div>
          ))}
        </main>
      </div>
    </div>
  );
}
```

### Product Grid — `src/components/product/product-grid.tsx`

```tsx
import type { Product } from "@/data-layer/types/product";
import { ProductCard } from "./product-card";

interface ProductGridProps {
  products: Product[];
}

export function ProductGrid({ products }: ProductGridProps) {
  if (products.length === 0) {
    return (
      <div className="text-center py-12 text-muted-foreground">
        <p className="text-lg">No products found</p>
        <p className="text-sm mt-1">Try adjusting your filters</p>
      </div>
    );
  }

  return (
    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
      {products.map((product) => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

### Product Card — `src/components/product/product-card.tsx`

```tsx
import Image from "next/image";
import Link from "next/link";
import type { Product } from "@/data-layer/types/product";
import { Badge } from "@/components/ui/badge";
import { formatPrice } from "@/lib/utils";

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  const image = product.images?.[0];
  const hasDiscount = product.compareAtPrice && product.compareAtPrice > product.price;

  return (
    <Link href={`/products/${product.slug}`} className="group block space-y-3">
      <div className="relative aspect-square overflow-hidden rounded-lg bg-muted">
        {image ? (
          <Image
            src={image.url}
            alt={image.alt}
            fill
            sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
            className="object-cover transition-transform group-hover:scale-105"
          />
        ) : (
          <div className="flex h-full items-center justify-center text-muted-foreground">
            No image
          </div>
        )}
        {!product.inStock && (
          <Badge variant="destructive" className="absolute top-2 right-2">Out of stock</Badge>
        )}
        {hasDiscount && (
          <Badge className="absolute top-2 left-2">Sale</Badge>
        )}
      </div>

      <div>
        <h3 className="font-medium group-hover:underline">{product.name}</h3>
        <div className="flex items-center gap-2 mt-1">
          <span className="font-semibold">{formatPrice(product.price)}</span>
          {hasDiscount && (
            <span className="text-sm text-muted-foreground line-through">
              {formatPrice(product.compareAtPrice!)}
            </span>
          )}
        </div>
        {product.categories[0] && (
          <p className="text-sm text-muted-foreground mt-1">{product.categories[0]}</p>
        )}
      </div>
    </Link>
  );
}
```

### Product Detail Page — `src/app/(shop)/products/[slug]/page.tsx`

> Server Component with `generateMetadata` for SEO. Uses canonical `Product` type — works with any active backend.

```tsx
import { Metadata } from "next";
import { notFound } from "next/navigation";
import { getProductBySlug } from "@/services/product.service";
import { ProductGallery } from "@/components/product/product-gallery";
import { ProductInfo } from "@/components/product/product-info";

interface ProductPageProps {
  params: Promise<{ slug: string }>;
}

export async function generateMetadata({ params }: ProductPageProps): Promise<Metadata> {
  const { slug } = await params;
  const product = await getProductBySlug(slug);

  if (!product) return { title: "Product Not Found" };

  return {
    title: product.name,
    description: product.description.slice(0, 160),
    openGraph: {
      title: product.name,
      description: product.description,
      images: product.images.map((img) => ({
        url: img.url,
        width: img.width,
        height: img.height,
        alt: img.alt,
      })),
    },
  };
}

export default async function ProductPage({ params }: ProductPageProps) {
  const { slug } = await params;
  const product = await getProductBySlug(slug);

  if (!product) notFound();

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="grid grid-cols-1 md:grid-cols-2 gap-12">
        <ProductGallery images={product.images} />
        <ProductInfo product={product} />
      </div>
    </div>
  );
}
```

### Product Detail Loading — `src/app/(shop)/products/[slug]/loading.tsx`

```tsx
export default function ProductDetailLoading() {
  return (
    <div className="container mx-auto px-4 py-8">
      <div className="grid grid-cols-1 md:grid-cols-2 gap-12">
        <div className="aspect-square bg-muted animate-pulse rounded-lg" />
        <div className="space-y-4">
          <div className="h-8 w-3/4 bg-muted animate-pulse rounded" />
          <div className="h-6 w-1/4 bg-muted animate-pulse rounded" />
          <div className="space-y-2 mt-6">
            {Array.from({ length: 4 }).map((_, i) => (
              <div key={i} className="h-4 bg-muted animate-pulse rounded" />
            ))}
          </div>
          <div className="h-12 w-full bg-muted animate-pulse rounded mt-8" />
        </div>
      </div>
    </div>
  );
}
```

### Product Gallery — `src/components/product/product-gallery.tsx`

```tsx
"use client";

import { useState } from "react";
import Image from "next/image";
import { cn } from "@/lib/utils";
import type { ProductImage } from "@/data-layer/types/product";

interface ProductGalleryProps {
  images: ProductImage[];
}

export function ProductGallery({ images }: ProductGalleryProps) {
  const [selectedIndex, setSelectedIndex] = useState(0);

  if (images.length === 0) {
    return (
      <div className="aspect-square bg-muted rounded-lg flex items-center justify-center text-muted-foreground">
        No images available
      </div>
    );
  }

  const selectedImage = images[selectedIndex];

  return (
    <div className="space-y-4">
      <div className="relative aspect-square overflow-hidden rounded-lg bg-muted">
        <Image
          src={selectedImage.url}
          alt={selectedImage.alt}
          fill
          sizes="(max-width: 768px) 100vw, 50vw"
          className="object-cover"
          priority
        />
      </div>

      {images.length > 1 && (
        <div className="flex gap-2 overflow-x-auto">
          {images.map((image, index) => (
            <button
              key={index}
              onClick={() => setSelectedIndex(index)}
              className={cn(
                "relative h-20 w-20 shrink-0 overflow-hidden rounded-md border-2",
                index === selectedIndex
                  ? "border-primary"
                  : "border-transparent opacity-70 hover:opacity-100"
              )}
            >
              <Image src={image.url} alt={image.alt} fill sizes="80px" className="object-cover" />
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

### Product Info — `src/components/product/product-info.tsx`

```tsx
import type { Product } from "@/data-layer/types/product";
import { Badge } from "@/components/ui/badge";
import { AddToCartButton } from "./add-to-cart-button";
import { formatPrice } from "@/lib/utils";

interface ProductInfoProps {
  product: Product;
}

export function ProductInfo({ product }: ProductInfoProps) {
  const hasDiscount = product.compareAtPrice && product.compareAtPrice > product.price;
  const discountPercent = hasDiscount
    ? Math.round((1 - product.price / product.compareAtPrice!) * 100)
    : 0;

  return (
    <div className="flex flex-col">
      {product.categories[0] && (
        <p className="text-sm text-muted-foreground mb-2">{product.categories[0]}</p>
      )}

      <h1 className="text-3xl font-bold">{product.name}</h1>

      <div className="flex items-center gap-3 mt-4">
        <span className="text-2xl font-bold">{formatPrice(product.price)}</span>
        {hasDiscount && (
          <>
            <span className="text-lg text-muted-foreground line-through">
              {formatPrice(product.compareAtPrice!)}
            </span>
            <Badge variant="destructive">{discountPercent}% off</Badge>
          </>
        )}
      </div>

      <div className="mt-2">
        {product.inStock ? (
          <Badge variant="outline" className="text-green-600 border-green-600">In stock</Badge>
        ) : (
          <Badge variant="destructive">Out of stock</Badge>
        )}
      </div>

      {product.description && (
        <p className="text-muted-foreground mt-6 leading-relaxed">{product.description}</p>
      )}

      <div className="mt-8">
        <AddToCartButton productId={product.id} disabled={!product.inStock} />
      </div>
    </div>
  );
}
```

### Add to Cart Button — `src/components/product/add-to-cart-button.tsx`

> Uses a Server Action (`addToCartAction`) — no GraphQL hooks, no Apollo, backend-agnostic.

```tsx
"use client";

import { useState, useTransition } from "react";
import { addToCartAction } from "@/actions/cart.actions";
import { useCartStore } from "@/stores/cart.store";
import { Button } from "@/components/ui/button";

interface AddToCartButtonProps {
  productId: string;
  variantId?: string;
  disabled?: boolean;
}

export function AddToCartButton({ productId, variantId, disabled }: AddToCartButtonProps) {
  const [quantity, setQuantity] = useState(1);
  const [isPending, startTransition] = useTransition();
  const cartId = useCartStore((s) => s.cartId);

  function handleAddToCart() {
    startTransition(async () => {
      await addToCartAction(cartId, productId, quantity, variantId);
    });
  }

  return (
    <div className="flex items-center gap-4">
      <div className="flex items-center border rounded-md">
        <button
          className="px-3 py-2 hover:bg-muted"
          onClick={() => setQuantity((q) => Math.max(1, q - 1))}
        >
          -
        </button>
        <span className="px-4 py-2 min-w-[3rem] text-center">{quantity}</span>
        <button
          className="px-3 py-2 hover:bg-muted"
          onClick={() => setQuantity((q) => q + 1)}
        >
          +
        </button>
      </div>

      <Button
        size="lg"
        className="flex-1"
        disabled={disabled || isPending}
        onClick={handleAddToCart}
      >
        {isPending ? "Adding..." : disabled ? "Out of Stock" : "Add to Cart"}
      </Button>
    </div>
  );
}
```

### Product Filters — `src/components/product/product-filters.tsx`

> Categories are fetched server-side and passed as props — no GraphQL hook needed in the client component.

```tsx
"use client";

import { useRouter, useSearchParams } from "next/navigation";
import { cn } from "@/lib/utils";
import { Input } from "@/components/ui/input";
import { Button } from "@/components/ui/button";

interface Category {
  id: string;
  name: string;
  slug: string;
}

interface ProductFiltersProps {
  categories: Category[];
  activeCategory?: string;
  minPrice?: string;
  maxPrice?: string;
}

export function ProductFilters({ categories, activeCategory, minPrice, maxPrice }: ProductFiltersProps) {
  const router = useRouter();
  const searchParams = useSearchParams();

  function updateFilter(key: string, value: string | undefined) {
    const params = new URLSearchParams(searchParams.toString());
    if (value) {
      params.set(key, value);
    } else {
      params.delete(key);
    }
    params.delete("page");
    router.push(`/products?${params.toString()}`);
  }

  return (
    <div className="space-y-6">
      <div>
        <h3 className="font-semibold mb-3">Categories</h3>
        <ul className="space-y-1">
          <li>
            <button
              onClick={() => updateFilter("category", undefined)}
              className={cn("text-sm hover:underline", !activeCategory && "font-bold")}
            >
              All Products
            </button>
          </li>
          {categories.map((cat) => (
            <li key={cat.id}>
              <button
                onClick={() => updateFilter("category", cat.slug)}
                className={cn("text-sm hover:underline", activeCategory === cat.slug && "font-bold")}
              >
                {cat.name}
              </button>
            </li>
          ))}
        </ul>
      </div>

      <div>
        <h3 className="font-semibold mb-3">Price Range</h3>
        <div className="flex gap-2">
          <Input
            type="number"
            placeholder="Min"
            defaultValue={minPrice}
            onBlur={(e) => updateFilter("minPrice", e.target.value || undefined)}
          />
          <Input
            type="number"
            placeholder="Max"
            defaultValue={maxPrice}
            onBlur={(e) => updateFilter("maxPrice", e.target.value || undefined)}
          />
        </div>
      </div>

      {(activeCategory || minPrice || maxPrice) && (
        <Button variant="outline" size="sm" className="w-full" onClick={() => router.push("/products")}>
          Clear All Filters
        </Button>
      )}
    </div>
  );
}
```

### Cart Store — `src/stores/cart.store.ts`

```ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface CartState {
  cartId: string;
  setCartId: (id: string) => void;
}

export const useCartStore = create<CartState>()(
  persist(
    (set) => ({
      cartId: "",
      setCartId: (cartId) => set({ cartId }),
    }),
    { name: "cart-storage" }
  )
);
```

### Utility: formatPrice — `src/lib/utils.ts`

> Add this to your existing `utils.ts` alongside `cn()` and `formatDate()`.

```ts
export function formatPrice(price: number, currency = "USD"): string {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency,
  }).format(price);
}
```

---

### Tailwind CSS v4 — `src/styles/globals.css`

> **Tailwind v4:** No more `tailwind.config.js`. Configuration is CSS-native using `@import` and `@theme`. Content detection is automatic.

```css
@import "tailwindcss";

@theme {
  --color-primary: #3b82f6;
  --color-primary-foreground: #ffffff;
  --color-destructive: #ef4444;
  --color-destructive-foreground: #ffffff;
  --color-accent: #f1f5f9;
  --color-accent-foreground: #0f172a;
  --color-background: #ffffff;
  --color-foreground: #0f172a;
  --color-input: #e2e8f0;

  --font-sans: "Inter", sans-serif;
  --radius-md: 0.375rem;
}

@layer base {
  * {
    @apply border-input;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

---

## Best Practices

### 1. Use the `src/` Directory

Keep the root clean with only config files. All application code goes inside `src/` for clear separation between code and configuration.

### 2. App Router + Server Components

Default to Server Components for everything. Only add `"use client"` when you need interactivity, hooks, or browser APIs. This reduces bundle size and improves performance.

### 3. Server Actions for Mutations

Use Server Actions (`"use server"`) for form submissions and data mutations instead of API routes. Pair with `useActionState` (React 19) for form state management and `useOptimistic` for instant UI feedback.

### 4. Route Groups with Parentheses

Use `(auth)`, `(dashboard)`, `(marketing)` to share layouts across pages without affecting the URL structure. This keeps your routing clean and your layouts organized.

### 5. Backend Adapter Pattern

Never import a GraphQL client, fetch, or any backend SDK directly in services or components. All data access goes through `getBackend()` which returns a `BackendAdapter` typed to your repository interfaces. Define canonical domain types in `src/data-layer/types/` — these are the only types your app cares about. Each adapter in `src/data-layer/adapters/` implements the interfaces and handles all normalization internally. Switch backends by changing `BACKEND_PROVIDER` in `.env.local` — zero changes to services, actions, or components. GraphQL adapters (Magento, Shopify) use `graphql-request` (~5KB) with inline `gql.tada` queries — no codegen step, no `generated/` folder. REST adapters (Odoo, custom) use native `fetch` with Next.js cache integration.

### 6. Services Layer

Extract business logic into `services/`. Components should handle UI only, while services handle data fetching, transformations, and business rules. This makes logic testable and reusable.

### 7. Shared Zod Validations

Define validation schemas once in `lib/validations/` and reuse them in both client-side forms, Server Actions, and API route handlers. This gives you full-stack type safety from a single source of truth.

### 8. Barrel Exports

Use `index.ts` files to create clean public APIs for each module. Only re-export what consumers need, keeping internal implementation details private.

### 9. Environment Variable Safety

Prefix client-side variables with `NEXT_PUBLIC_`. Note that `NEXT_PUBLIC_*` vars are baked in at **build time** — they cannot be changed at runtime. Keep `.env.example` committed so new developers know which variables to configure. Never commit `.env.local`.

### 10. Error and Loading Boundaries

Add `error.tsx` and `loading.tsx` at every meaningful route segment. Add `global-error.tsx` at the root to catch errors in the root layout. This provides graceful degradation and instant loading feedback throughout your app.

### 11. Path Aliases

Configure `@/` aliases in `tsconfig.json`. Never write `../../../components` — always use `@/components`. This keeps imports clean and refactoring painless.

### 12. Defense-in-Depth Auth

Never rely solely on the proxy for authentication. Always verify sessions in Server Components and Route Handlers too. Use Auth.js v5's `auth()` function in dashboard layouts and sensitive API routes.

### 13. Proxy, Not Middleware

Next.js 16 renames `middleware.ts` to `proxy.ts` (with the exported function renamed to `proxy`). The proxy runs on **Node.js runtime only** — Edge Runtime is no longer supported. Use it as a last resort for request-level concerns (redirects, headers). Prefer route handlers, server components, and `next.config.ts` redirects when possible.

### 14. Async Dynamic APIs

`cookies()`, `headers()`, `params`, and `searchParams` are async Promises that must be `await`ed (introduced in Next.js 15, fully enforced in 16). This enables streaming and partial prerendering.

### 15. Co-located Unit Tests + Separate E2E

Place unit tests (`*.test.ts` / `*.test.tsx`) next to the source file they test. This makes it easy to spot untested files and keeps moves/renames in sync. E2E tests (`*.spec.ts`) stay in `tests/e2e/` since they test full user flows, not single files. Shared test fixtures live in `tests/fixtures/`. Use Playwright's locator-based API (`getByRole`, `getByLabel`) for resilient E2E tests.

### 16. Docker Multi-Stage Builds

Use `output: "standalone"` in Next.js config combined with multi-stage Dockerfiles. This produces minimal production images around 150MB. Remember that `NEXT_PUBLIC_`* vars must be set at build time.

### 17. CI/CD Pipeline

Automate the quality gates: lint → type-check → test → build on every pull request. Block merges when any step fails.

### 18. Performance — Google PageSpeed Optimization

The architecture is optimized for Core Web Vitals (LCP, CLS, INP):

- **LCP (Largest Contentful Paint):** Use `next/image` with `priority` on above-the-fold hero/product images. Use `<link rel="preconnect">` for the backend API origin (configured per adapter in `src/config/site.ts`). Self-host fonts via `next/font` to eliminate render-blocking requests.
- **CLS (Cumulative Layout Shift):** Always specify `width` and `height` on images. Use `loading.tsx` skeletons that match final layout dimensions. Avoid client-side-only content that shifts the page.
- **INP (Interaction to Next Paint):** Use `next/dynamic` with `ssr: false` for heavy client components (product gallery lightbox, rich text editors). Use `useOptimistic` for instant cart/filter interactions.
- **Caching:** Set `revalidate` on product/category pages for ISR. Use `React.cache()` to deduplicate server-side data fetches. Configure `Cache-Control` headers for static assets.
- **Bundle size:** Use `@next/bundle-analyzer` to audit dependencies. Lazy-load non-critical components with `next/dynamic`. `graphql-request` (~5KB) + `gql.tada` (~1KB) replaces Apollo Client (~40KB) across all GraphQL adapters. Only the active adapter's client is loaded at runtime.
- **Monitoring:** Report Web Vitals to your analytics provider via `lib/web-vitals.ts`. Track per-page, per-locale, and per-store metrics to catch regressions.

### 19. i18n — Locale-Aware Routing

All routes are nested under `[locale]/`. The proxy detects locale from URL → cookie → `Accept-Language` header → store default. Use `next-intl` for server/client translations. Set `<html lang>` and `dir="rtl"` dynamically per locale. Generate `alternates.languages` in metadata for SEO hreflang tags.

### 20. Multi-Currency

Currency is managed via Zustand (`currency.store.ts`) and formatted with `Intl.NumberFormat` in `lib/currency.ts`. Exchange rates are fetched and cached in `services/currency.service.ts`. Each store defines a `defaultCurrency` and `supportedCurrencies`. The checkout flow passes the selected currency to Stripe.

### 21. Multi-Store

Stores are resolved from the request hostname in `proxy.ts`. Each store (`config/stores.ts`) defines its region, domain, default locale, default currency, supported locales/currencies, and optional theme overrides. Store context is available via `stores/store-config.store.ts` and the `useStoreConfig()` hook.

---

## Recommended Tech Stack


| Category    | Tool                     | Purpose                                        |
| ----------- | ------------------------ | ---------------------------------------------- |
| **Core**    | Next.js 16+              | React framework with App Router                |
|             | React 19+                | Server Components, Server Actions, `use()` API |
|             | TypeScript 5.5+          | Static typing with strict mode                 |
| **Styling** | Tailwind CSS 4           | Utility-first CSS (CSS-native config)          |
|             | shadcn/ui                | Composable, accessible components              |
|             | CVA                      | Component variant APIs                         |
| **Backend** | Adapter Pattern          | Swap Magento / Shopify / Odoo / custom via env var |
|             | graphql-request          | Minimal (~5KB) server-side GraphQL client      |
|             | gql.tada                 | Inline GraphQL queries with inferred types (no codegen) |
| **i18n**    | next-intl                | Server/client translations, locale routing     |
| **Data**    | Zustand                  | Lightweight client-side UI + currency + store state |
| **Auth**    | Auth.js v5 (NextAuth v5) | OAuth, magic links, credentials                |
|             | Zod                      | Runtime schema validation                      |
| **Forms**   | `useActionState`         | React 19 form state with Server Actions        |
|             | `useFormStatus`          | Submission status without prop drilling        |
|             | `useOptimistic`          | Instant UI feedback during mutations           |
| **Testing** | Vitest                   | Fast unit testing                              |
|             | Playwright               | Cross-browser E2E testing (locator API)        |
|             | ESLint 9 + Prettier      | Code quality (flat config) & formatting        |
| **Perf**    | `@next/bundle-analyzer`  | Audit bundle size & dependency tree            |
|             | `web-vitals`             | Core Web Vitals monitoring (CLS, LCP, INP)     |
| **DevOps**  | Docker                   | Multi-stage production builds                  |
|             | GitHub Actions           | CI/CD pipeline                                 |
|             | Vercel / AWS             | Deployment with edge functions                 |


---

## Backend Integration Guide

### Step 1: Choose Your Backend

Set `BACKEND_PROVIDER` in `.env.local` and add the matching credentials:

```env
# Choose one: magento | shopify | odoo | custom
BACKEND_PROVIDER="magento"

# Then add only the vars for your chosen provider (see .env.example)
MAGENTO_GRAPHQL_URL="https://your-magento.com/graphql"
MAGENTO_ADMIN_TOKEN="your-token"
```

### Step 2: Install Dependencies for Your Backend

```bash
# For any GraphQL backend (Magento, Shopify)
pnpm add graphql-request gql.tada graphql
pnpm add -D @0no-co/graphqlsp              # editor LSP plugin for inline type inference

# For REST backends (Odoo, custom) — no extra deps needed
# Uses native fetch built into Next.js
```

### Step 3: Configure gql.tada (GraphQL backends only)

`gql.tada` infers types from inline queries — no codegen step, no `generated/` folder. It only needs the schema introspected **once** per adapter.

**a.** Add an LSP config to `tsconfig.json`:

```json
{
  "compilerOptions": {
    "plugins": [
      {
        "name": "@0no-co/graphqlsp",
        "schemas": [
          { "name": "magento", "schema": "./schemas/magento.graphql", "tadaOutputLocation": "./schemas/magento-env.d.ts" },
          { "name": "shopify", "schema": "./schemas/shopify.graphql", "tadaOutputLocation": "./schemas/shopify-env.d.ts" }
        ]
      }
    ]
  }
}
```

**b.** Fetch the schemas (one-off, re-run when schemas change):

```json
{
  "scripts": {
    "schema:magento": "gql.tada generate-schema $MAGENTO_GRAPHQL_URL --output schemas/magento.graphql",
    "schema:shopify": "gql.tada generate-schema $SHOPIFY_STOREFRONT_URL --output schemas/shopify.graphql --header \"X-Shopify-Storefront-Access-Token: $SHOPIFY_STOREFRONT_TOKEN\""
  }
}
```

```bash
pnpm schema:magento    # fetch Magento schema once
pnpm schema:shopify    # fetch Shopify schema once
```

That's it — queries written with `graphql(\`...\`)` are now fully typed via the LSP. No build step, no watch mode, no generated code to commit. REST adapters skip this step entirely.

### Step 4: Implement Your Adapter (custom backends)

Copy `src/data-layer/adapters/custom/` and implement the three repository interfaces:

```ts
// src/data-layer/adapters/my-backend/product.adapter.ts
import type { IProductRepository } from "@/data-layer/interfaces/product.repository";

export class MyBackendProductAdapter implements IProductRepository {
  async getProducts(filters) {
    const res = await fetch(`${process.env.CUSTOM_API_URL}/products`);
    const data = await res.json();
    return {
      items: data.results.map(normalize),   // map to canonical Product type
      total: data.count,
      page: 1, limit: 20, hasNextPage: false,
    };
  }
  // ... implement remaining methods
}
```

### Step 5: Register the Adapter

Export your adapter from its `index.ts` and add it to the factory:

```ts
// src/data-layer/adapters/my-backend/index.ts
import { MyBackendProductAdapter } from "./product.adapter";
export const MyBackendAdapter = {
  products: new MyBackendProductAdapter(),
  cart: new MyBackendCartAdapter(),
  users: new MyBackendUserAdapter(),
};

// src/data-layer/factory.ts — add a new case
case "my-backend":
  _backend = require("./adapters/my-backend").MyBackendAdapter;
  break;
```

### Step 6: Use in Services and Actions

All services and server actions call `getBackend()` — they work identically regardless of which adapter is active:

```ts
// services and actions never change when you switch backends
const { products, cart, users } = getBackend();
```

> **Tip:** With `gql.tada`, there's nothing to watch or regenerate during development — types are inferred at edit time by the `@0no-co/graphqlsp` LSP plugin. Re-run `pnpm schema:magento` or `pnpm schema:shopify` only when the upstream GraphQL schema changes.

---

## Quick Start

```bash
# Clone and install
git clone <repo-url> my-enterprise-app
cd my-enterprise-app
pnpm install

# Setup environment
cp .env.example .env.local
# Edit .env.local — set BACKEND_PROVIDER and the matching backend vars

# Fetch GraphQL schema (one-off — only for GraphQL backends)
pnpm schema:magento    # Introspects Magento schema into ./schemas/magento.graphql
pnpm schema:shopify    # Introspects Shopify schema into ./schemas/shopify.graphql
# After this, queries are typed inline via gql.tada — no build step needed.
# REST adapters (Odoo, custom) skip this step entirely.

# Development
pnpm dev               # Start dev server at localhost:3000

# Quality checks
pnpm lint              # ESLint
pnpm type-check        # TypeScript compiler check
pnpm test              # Unit tests (Vitest)
pnpm test:e2e          # E2E tests (Playwright)

# Production
pnpm build             # Build for production
pnpm start             # Start production server

# Docker
docker compose up      # Local dev with Redis
docker build -f docker/Dockerfile -t myapp .
```

