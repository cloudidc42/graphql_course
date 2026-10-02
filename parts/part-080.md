# Part 080 — GraphQL Client Patterns: React & Beyond ⚛️

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 2091–2130

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Apollo Client advanced patterns
- React Query + GraphQL
- TanStack Query integration
- Zustand กับ GraphQL
- urql client
- SWR กับ GraphQL
- Optimistic updates patterns
- Normalized cache strategies

---

## 📌 Step 2091: Apollo Client Advanced

```typescript
// Apollo Client: advanced setup
import {
  ApolloClient,
  InMemoryCache,
  HttpLink,
  split,
  from,
} from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { createClient } from 'graphql-ws';
import { getMainDefinition } from '@apollo/client/utilities';
import { RetryLink } from '@apollo/client/link/retry';
import { onError } from '@apollo/client/link/error';

// Error handling link
const errorLink = onError(({ graphQLErrors, networkError, operation, forward }) => {
  if (graphQLErrors) {
    for (const err of graphQLErrors) {
      switch (err.extensions?.code) {
        case 'UNAUTHENTICATED':
          // Redirect to login
          window.location.href = '/login';
          break;
        case 'RATE_LIMITED':
          // Back off and retry
          return forward(operation);
      }
    }
  }
  
  if (networkError) {
    console.error('Network error:', networkError);
  }
});

// Retry link สำหรับ network errors
const retryLink = new RetryLink({
  delay: {
    initial: 300,
    max: 5000,
    jitter: true,
  },
  attempts: {
    max: 3,
    retryIf: (error) => !!error && error.message !== 'Unauthorized',
  },
});

// HTTP link
const httpLink = new HttpLink({
  uri: process.env.NEXT_PUBLIC_GRAPHQL_URL,
  credentials: 'include',
});

// WebSocket link สำหรับ subscriptions
const wsLink = new GraphQLWsLink(
  createClient({
    url: process.env.NEXT_PUBLIC_WS_URL!,
    connectionParams: () => ({
      authorization: `Bearer ${getToken()}`,
    }),
    on: {
      connected: () => console.log('WS connected'),
      closed: () => console.log('WS closed'),
      error: (err) => console.error('WS error:', err),
    },
    shouldRetry: () => true,
    retryAttempts: Infinity,
    retryWait: async (retries) => {
      await new Promise(r => setTimeout(r, Math.min(1000 * 2 ** retries, 30000)));
    },
  })
);

// Split: queries/mutations → HTTP, subscriptions → WebSocket
const splitLink = split(
  ({ query }) => {
    const def = getMainDefinition(query);
    return def.kind === 'OperationDefinition' && def.operation === 'subscription';
  },
  wsLink,
  from([errorLink, retryLink, httpLink])
);

// Cache with normalized entities
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        products: {
          keyArgs: ['filter', 'orderBy'],
          merge(existing, incoming, { args }) {
            const existingEdges = existing?.edges ?? [];
            
            // After: append เมื่อมี cursor
            if (args?.after) {
              return {
                ...incoming,
                edges: [...existingEdges, ...incoming.edges],
              };
            }
            
            // Fresh fetch: replace
            return incoming;
          },
        },
      },
    },
    
    Product: {
      keyFields: ['id'],
    },
    
    User: {
      keyFields: ['id'],
      fields: {
        orders: {
          keyArgs: ['status'],
          merge: true,
        },
      },
    },
  },
});

export const client = new ApolloClient({
  link: splitLink,
  cache,
  defaultOptions: {
    watchQuery: {
      fetchPolicy: 'cache-and-network',
      nextFetchPolicy: 'cache-first',
    },
    query: {
      fetchPolicy: 'network-only',
    },
  },
});
```

---

## 📌 Step 2092: Custom Hooks Pattern

```typescript
// hooks/useProduct.ts
import { useQuery, useMutation, useApolloClient } from '@apollo/client';
import { GET_PRODUCT, UPDATE_PRODUCT, DELETE_PRODUCT } from '../graphql/product.operations';
import type { GetProductQuery, UpdateProductMutation } from '../generated/graphql';

export function useProduct(id: string) {
  const { data, loading, error, refetch } = useQuery<GetProductQuery>(GET_PRODUCT, {
    variables: { id },
    skip: !id,
    
    // Real-time: poll ทุก 30 วินาที
    // pollInterval: 30000,
  });
  
  const [updateProduct, { loading: updating }] = useMutation<UpdateProductMutation>(
    UPDATE_PRODUCT,
    {
      optimisticResponse: ({ id, input }) => ({
        __typename: 'Mutation',
        updateProduct: {
          __typename: 'Product',
          id,
          ...input,  // Optimistic data
        },
      }),
      
      onCompleted: () => {
        // Optional: show success notification
      },
      
      onError: (err) => {
        // Revert optimistic update automatically
        console.error('Update failed:', err);
      },
    }
  );
  
  const [deleteProduct] = useMutation(DELETE_PRODUCT, {
    update(cache, { data }) {
      // Remove from cache
      cache.evict({ id: cache.identify({ __typename: 'Product', id }) });
      cache.gc();
    },
  });
  
  return {
    product: data?.product,
    loading,
    error,
    updating,
    refetch,
    updateProduct: (input: UpdateProductInput) => 
      updateProduct({ variables: { id, input } }),
    deleteProduct: () => 
      deleteProduct({ variables: { id } }),
  };
}
```

---

## 📌 Step 2093: urql (Lightweight Alternative)

```bash
npm install urql graphql
```

```typescript
// urql: simpler, smaller alternative to Apollo Client
import { createClient, Provider, useQuery, useMutation, cacheExchange, fetchExchange } from 'urql';
import { subscriptionExchange } from '@urql/core';
import { createClient as createWsClient } from 'graphql-ws';

const wsClient = createWsClient({ url: 'ws://localhost:4000/graphql' });

const client = createClient({
  url: '/graphql',
  exchanges: [
    cacheExchange,       // Normalized cache
    fetchExchange,       // HTTP
    subscriptionExchange({
      forwardSubscription: (request) => ({
        subscribe: (sink) => ({
          unsubscribe: wsClient.subscribe(request, sink),
        }),
      }),
    }),
  ],
});

// Component using urql
function ProductCard({ id }: { id: string }) {
  const [{ data, fetching, error }] = useQuery({
    query: GET_PRODUCT,
    variables: { id },
  });
  
  const [, likeProduct] = useMutation(LIKE_PRODUCT);
  
  if (fetching) return <Skeleton />;
  if (error) return <Error message={error.message} />;
  
  return (
    <div>
      <h2>{data?.product?.name}</h2>
      <p>{data?.product?.price}</p>
      <button onClick={() => likeProduct({ id })}>Like</button>
    </div>
  );
}
```

---

## 📌 Step 2094: TanStack Query + GraphQL

```bash
npm install @tanstack/react-query graphql-request
```

```typescript
// TanStack Query: ไม่ต้องการ GraphQL client ตัวเต็ม
// เหมาะสำหรับ: simple apps, server components ที่ไม่ต้องการ normalized cache

import { QueryClient, QueryClientProvider, useQuery, useMutation } from '@tanstack/react-query';
import { request, gql } from 'graphql-request';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,  // 5 นาที
      retry: 2,
    },
  },
});

const GRAPHQL_URL = '/graphql';

// Typed fetcher
async function gqlFetch<T>(
  query: string,
  variables?: Record<string, unknown>
): Promise<T> {
  return request<T>(GRAPHQL_URL, query, variables, {
    Authorization: `Bearer ${getToken()}`,
  });
}

// Hooks
export function useProduct(id: string) {
  return useQuery({
    queryKey: ['product', id],
    queryFn: () => gqlFetch<{ product: Product }>(
      gql`query GetProduct($id: ID!) { product(id: $id) { id name price } }`,
      { id }
    ),
    select: (data) => data.product,
    enabled: !!id,
  });
}

export function useCreateProduct() {
  return useMutation({
    mutationFn: (input: CreateProductInput) => 
      gqlFetch<{ createProduct: Product }>(
        gql`mutation CreateProduct($input: CreateProductInput!) {
          createProduct(input: $input) { id name price }
        }`,
        { input }
      ),
    onSuccess: () => {
      // Invalidate products list
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

---

## 📌 Step 2095: สรุป Part 080

### เนื้อหาที่เรียนรู้

✅ Apollo Client advanced setup  
✅ Custom hooks pattern  
✅ urql (lightweight alternative)  
✅ TanStack Query + graphql-request  
✅ Optimistic updates  
✅ Cache policies  

### Client Library Comparison

```
Apollo Client:
✅ Full-featured: normalized cache, optimistic, subscriptions
✅ Apollo Studio integration
❌ Large bundle (~50KB)
→ ใช้สำหรับ: complex apps, dashboard

urql:
✅ Smaller (~15KB)
✅ Flexible exchanges
✅ Simple API
→ ใช้สำหรับ: medium apps

TanStack Query + graphql-request:
✅ Smallest, familiar
✅ No GraphQL-specific cache
❌ No normalized cache
→ ใช้สำหรับ: simple apps, SSR

Server Components (Next.js App Router):
✅ ไม่ต้องการ client library
✅ Fetch ตรงใน component
→ ใช้สำหรับ: static/SSR content
```

### ในส่วนถัดไป

➡️ **[Part 081](./part-081.md)** — GraphQL Monitoring & Alerting

---

*Part 080 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
