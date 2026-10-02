# Part 054 — Server-Side Rendering with Next.js 🖥️

> **ระดับ:** Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1071–1110

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Apollo Client ใน Next.js App Router
- Server Components + GraphQL
- SSR (getServerSideProps) pattern
- SSG (getStaticProps) pattern
- ISR (Incremental Static Regeneration)
- Streaming + Suspense
- Hydration strategies
- next/headers สำหรับ auth cookies

---

## 📌 Step 1071: Next.js App Router Setup

```bash
npx create-next-app@latest graphql-nextjs --typescript --app --tailwind
cd graphql-nextjs
npm install @apollo/client @apollo/experimental-nextjs-app-support graphql
```

```typescript
// src/lib/apollo-client.ts
import { ApolloClient, HttpLink, InMemoryCache, from } from '@apollo/client';
import { onError } from '@apollo/client/link/error';
import { setContext } from '@apollo/client/link/context';

const errorLink = onError(({ graphQLErrors, networkError }) => {
  if (graphQLErrors) {
    graphQLErrors.forEach(({ message, extensions }) => {
      if (extensions?.code === 'UNAUTHENTICATED') {
        // Redirect to login
        if (typeof window !== 'undefined') {
          window.location.href = '/login';
        }
      }
    });
  }
});

const httpLink = new HttpLink({
  uri: process.env.NEXT_PUBLIC_GRAPHQL_URL ?? 'http://localhost:4000/graphql',
  credentials: 'include', // ส่ง cookies
});

const authLink = setContext((_, { headers }) => {
  const token = typeof window !== 'undefined'
    ? localStorage.getItem('accessToken')
    : null;
  
  return {
    headers: {
      ...headers,
      Authorization: token ? `Bearer ${token}` : '',
    },
  };
});

export function makeClient() {
  return new ApolloClient({
    cache: new InMemoryCache({
      typePolicies: {
        Product: { keyFields: ['id'] },
        User: { keyFields: ['id'] },
      },
    }),
    link: from([errorLink, authLink, httpLink]),
    defaultOptions: {
      watchQuery: { fetchPolicy: 'cache-and-network' },
    },
  });
}
```

---

## 📌 Step 1072: Apollo Provider สำหรับ App Router

```typescript
// src/lib/apollo-wrapper.tsx — สำหรับ App Router
'use client';

import { ApolloNextAppProvider, ApolloClient, InMemoryCache } from '@apollo/experimental-nextjs-app-support';
import { makeClient } from './apollo-client.js';

export function ApolloWrapper({ children }: { children: React.ReactNode }) {
  return (
    <ApolloNextAppProvider makeClient={makeClient}>
      {children}
    </ApolloNextAppProvider>
  );
}

// src/app/layout.tsx
import { ApolloWrapper } from '../lib/apollo-wrapper';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="th">
      <body>
        <ApolloWrapper>
          {children}
        </ApolloWrapper>
      </body>
    </html>
  );
}
```

---

## 📌 Step 1073: Server Component + GraphQL

```typescript
// src/app/products/page.tsx
// React Server Component — รัน บน server เท่านั้น
// ไม่ต้องการ Apollo Client ใน client bundle

import { getClient } from '../lib/server-apollo-client';
import { gql } from '@apollo/client';

const GET_PRODUCTS = gql`
  query GetProducts($first: Int, $filter: ProductFilter) {
    products(first: $first, filter: $filter) {
      edges {
        node {
          id
          name
          price
          image { url }
          category { name }
        }
      }
      pageInfo { hasNextPage endCursor }
      totalCount
    }
  }
`;

// Server Component: async function ได้เลย
export default async function ProductsPage({
  searchParams,
}: {
  searchParams: { category?: string; page?: string };
}) {
  const client = getClient();
  
  const { data, error } = await client.query({
    query: GET_PRODUCTS,
    variables: {
      first: 20,
      filter: {
        categoryId: searchParams.category ?? undefined,
      },
    },
  });
  
  if (error) {
    throw new Error(error.message);
  }
  
  const products = data.products.edges.map(e => e.node);
  
  return (
    <main>
      <h1>Products ({data.products.totalCount})</h1>
      <div className="grid grid-cols-4 gap-4">
        {products.map(product => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </main>
  );
}

// Server Apollo Client (per-request)
// src/lib/server-apollo-client.ts
import { ApolloClient, HttpLink, InMemoryCache } from '@apollo/client';
import { cookies } from 'next/headers';
import { registerApolloClient } from '@apollo/experimental-nextjs-app-support/rsc';

export const { getClient } = registerApolloClient(() => {
  const cookieStore = cookies();
  const token = cookieStore.get('accessToken')?.value;
  
  return new ApolloClient({
    cache: new InMemoryCache(),
    link: new HttpLink({
      uri: process.env.GRAPHQL_INTERNAL_URL ?? 'http://graphql-server:4000/graphql',
      headers: {
        Authorization: token ? `Bearer ${token}` : '',
      },
    }),
  });
});
```

---

## 📌 Step 1074: SSG กับ getStaticProps

```typescript
// pages/products/[id].tsx — Pages Router (ยังใช้ได้)

import { GetStaticProps, GetStaticPaths } from 'next';
import { ApolloClient, InMemoryCache, gql } from '@apollo/client';

const GET_PRODUCT = gql`
  query GetProduct($id: ID!) {
    product(id: $id) {
      id name description price
      images { url alt }
      category { id name }
      variants { id name price stock }
    }
  }
`;

const GET_ALL_PRODUCT_IDS = gql`
  query GetAllProductIds {
    products(first: 1000) {
      edges { node { id } }
    }
  }
`;

export const getStaticPaths: GetStaticPaths = async () => {
  const client = new ApolloClient({
    uri: process.env.GRAPHQL_URL,
    cache: new InMemoryCache(),
  });
  
  const { data } = await client.query({ query: GET_ALL_PRODUCT_IDS });
  
  return {
    paths: data.products.edges.map(e => ({
      params: { id: e.node.id },
    })),
    fallback: 'blocking', // สร้างหน้าใหม่ on-demand ถ้าไม่มี
  };
};

export const getStaticProps: GetStaticProps = async ({ params }) => {
  const client = new ApolloClient({
    uri: process.env.GRAPHQL_URL,
    cache: new InMemoryCache(),
  });
  
  try {
    const { data } = await client.query({
      query: GET_PRODUCT,
      variables: { id: params?.id },
    });
    
    if (!data.product) {
      return { notFound: true };
    }
    
    return {
      props: {
        product: data.product,
        initialApolloState: client.cache.extract(),
      },
      revalidate: 3600, // ISR: re-generate ทุก 1 ชั่วโมง
    };
  } catch {
    return { notFound: true };
  }
};

// Component
export default function ProductPage({ product }) {
  return <ProductDetail product={product} />;
}
```

---

## 📌 Step 1075: Streaming กับ Suspense

```tsx
// App Router + Streaming
// src/app/dashboard/page.tsx

import { Suspense } from 'react';
import { getClient } from '../../lib/server-apollo-client';

// Skeleton loading
function StatsSkeleton() {
  return <div className="animate-pulse h-32 bg-gray-200 rounded" />;
}

function OrderSkeleton() {
  return (
    <div className="space-y-2">
      {Array.from({ length: 5 }).map((_, i) => (
        <div key={i} className="animate-pulse h-16 bg-gray-200 rounded" />
      ))}
    </div>
  );
}

// Async Server Components สำหรับ parallel fetching
async function DashboardStats() {
  const client = getClient();
  const { data } = await client.query({
    query: gql`query DashboardStats { stats { revenue orders users } }`,
  });
  
  return <StatsCards stats={data.stats} />;
}

async function RecentOrders() {
  const client = getClient();
  const { data } = await client.query({
    query: gql`query RecentOrders { orders(first: 5, orderBy: CREATED_AT_DESC) { edges { node { id total status createdAt } } } }`,
  });
  
  return <OrderList orders={data.orders.edges.map(e => e.node)} />;
}

async function TopProducts() {
  // simulate slow query
  const client = getClient();
  const { data } = await client.query({
    query: gql`query TopProducts { topProducts(limit: 5) { id name salesCount } }`,
  });
  
  return <ProductList products={data.topProducts} />;
}

// Page: render sections independently — stream as each resolves
export default function DashboardPage() {
  return (
    <div className="grid grid-cols-3 gap-6">
      {/* Stats loads first (fast query) */}
      <div className="col-span-3">
        <Suspense fallback={<StatsSkeleton />}>
          <DashboardStats />
        </Suspense>
      </div>
      
      {/* Orders and Products load in parallel */}
      <div className="col-span-2">
        <Suspense fallback={<OrderSkeleton />}>
          <RecentOrders />
        </Suspense>
      </div>
      
      <div className="col-span-1">
        <Suspense fallback={<div>Loading top products...</div>}>
          <TopProducts />
        </Suspense>
      </div>
    </div>
  );
}
```

---

## 📌 Step 1076: Authentication ใน Next.js

```typescript
// src/middleware.ts — Next.js Middleware สำหรับ auth guard

import { NextRequest, NextResponse } from 'next/server';
import { jwtVerify } from 'jose';

const PROTECTED_PATHS = ['/dashboard', '/orders', '/profile', '/admin'];
const AUTH_PATHS = ['/login', '/register'];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const isProtected = PROTECTED_PATHS.some(p => pathname.startsWith(p));
  const isAuthPath = AUTH_PATHS.some(p => pathname.startsWith(p));
  
  const token = request.cookies.get('accessToken')?.value;
  
  // ตรวจสอบ token validity
  let isValidToken = false;
  if (token) {
    try {
      await jwtVerify(token, new TextEncoder().encode(process.env.JWT_SECRET!));
      isValidToken = true;
    } catch {
      isValidToken = false;
    }
  }
  
  // Redirect ถ้าไม่ได้ login และพยายามเข้า protected path
  if (isProtected && !isValidToken) {
    const url = request.nextUrl.clone();
    url.pathname = '/login';
    url.searchParams.set('redirect', pathname);
    return NextResponse.redirect(url);
  }
  
  // Redirect ถ้า login แล้วพยายามเข้า auth paths
  if (isAuthPath && isValidToken) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }
  
  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/orders/:path*', '/profile/:path*', '/admin/:path*', '/login', '/register'],
};

// src/app/login/page.tsx — Login form
'use client';
import { useMutation } from '@apollo/client';

const LOGIN_MUTATION = gql`
  mutation Login($email: String!, $password: String!) {
    login(email: $email, password: $password) {
      ... on LoginSuccess {
        accessToken
        refreshToken
        user { id name }
      }
      ... on InvalidCredentialsError {
        message
        remainingAttempts
      }
    }
  }
`;

export default function LoginPage() {
  const [login, { loading }] = useMutation(LOGIN_MUTATION, {
    onCompleted: ({ login }) => {
      if (login.__typename === 'LoginSuccess') {
        // Set cookies
        document.cookie = `accessToken=${login.accessToken}; path=/; secure; samesite=strict`;
        router.push('/dashboard');
      }
    },
  });
  
  // ... form JSX
}
```

---

## 📌 Step 1077: สรุป Part 054

### เนื้อหาที่เรียนรู้

✅ Apollo Client ใน Next.js App Router  
✅ Server Components + GraphQL  
✅ SSG ด้วย getStaticProps  
✅ ISR (revalidate)  
✅ Streaming กับ Suspense  
✅ Next.js Middleware สำหรับ auth  

### Rendering Strategies สำหรับ GraphQL

```
Server Component (App Router):
✅ ไม่มี client bundle
✅ เข้าถึง secrets/headers ได้โดยตรง  
✅ SEO ดี
❌ ไม่มี interactivity

Client Component ('use client'):
✅ Real-time updates
✅ User interactions
✅ Apollo cache/optimistic updates
❌ ต้องการ client bundle

SSG + ISR:
✅ Performance สูงสุด (CDN cached)
✅ ดี สำหรับ content ที่เปลี่ยนไม่บ่อย
❌ ข้อมูลอาจ stale

SSR (per-request):
✅ Data fresh เสมอ
✅ Personalized content
❌ ช้ากว่า SSG
```

### ในส่วนถัดไป

➡️ **[Part 055](./part-055.md)** — GraphQL Mesh & Schema Unification

---

*Part 054 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
