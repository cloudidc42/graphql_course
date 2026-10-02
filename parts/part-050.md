# Part 050 — Relay-Style Cursor Pagination 📄

> **ระดับ:** Intermediate-Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 911–950

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Relay Connection Specification
- Cursor encoding/decoding
- Forward pagination (first/after)
- Backward pagination (last/before)
- PageInfo และ edge metadata
- Total count strategies
- Stable cursor design
- Filtering + sorting ร่วมกับ cursor pagination

---

## 📌 Step 911: Relay Connection Spec

```graphql
# Relay Connection Specification
# https://relay.dev/graphql/connections.htm

type Query {
  # Forward pagination: first + after
  products(
    first: Int
    after: String
    last: Int
    before: String
    filter: ProductFilter
    orderBy: ProductOrderBy
  ): ProductConnection!
}

type ProductConnection {
  edges: [ProductEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ProductEdge {
  node: Product!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

input ProductFilter {
  minPrice: Float
  maxPrice: Float
  categoryId: ID
  inStock: Boolean
  search: String
}

enum ProductOrderBy {
  CREATED_AT_ASC
  CREATED_AT_DESC
  PRICE_ASC
  PRICE_DESC
  NAME_ASC
  NAME_DESC
}
```

---

## 📌 Step 912: Cursor Encoding

```typescript
// src/pagination/cursor.ts

// Cursor คือ opaque string ที่ client ไม่ควร parse
// encode: position data → base64 string
// decode: base64 string → position data

interface CursorData {
  id: string;
  value?: string | number;  // สำหรับ sort value
  field?: string;           // sort field name
}

export function encodeCursor(data: CursorData): string {
  const json = JSON.stringify(data);
  return Buffer.from(json).toString('base64url');
}

export function decodeCursor(cursor: string): CursorData {
  try {
    const json = Buffer.from(cursor, 'base64url').toString('utf-8');
    const data = JSON.parse(json) as CursorData;
    if (!data.id) throw new Error('Invalid cursor: missing id');
    return data;
  } catch {
    throw new GraphQLError('Invalid cursor', {
      extensions: { code: 'BAD_USER_INPUT' }
    });
  }
}

// Simple cursor (id only)
export function encodeSimpleCursor(id: string): string {
  return Buffer.from(`cursor:${id}`).toString('base64url');
}

export function decodeSimpleCursor(cursor: string): string {
  const decoded = Buffer.from(cursor, 'base64url').toString('utf-8');
  if (!decoded.startsWith('cursor:')) throw new Error('Invalid cursor format');
  return decoded.slice(7);
}
```

---

## 📌 Step 913: Pagination Helper

```typescript
// src/pagination/paginate.ts
import { Prisma, PrismaClient } from '@prisma/client';
import { decodeCursor, encodeCursor } from './cursor.js';

interface PaginationArgs {
  first?: number | null;
  after?: string | null;
  last?: number | null;
  before?: string | null;
}

interface PaginationResult<T extends { id: string }> {
  edges: Array<{ node: T; cursor: string }>;
  pageInfo: {
    hasNextPage: boolean;
    hasPreviousPage: boolean;
    startCursor: string | null;
    endCursor: string | null;
  };
  totalCount: number;
}

// Validate pagination args
function validatePaginationArgs(args: PaginationArgs): void {
  const { first, last, after, before } = args;
  
  if (first != null && first < 0) throw new GraphQLError('first must be >= 0');
  if (last != null && last < 0) throw new GraphQLError('last must be >= 0');
  if (first != null && first > 100) throw new GraphQLError('first must be <= 100');
  if (last != null && last > 100) throw new GraphQLError('last must be <= 100');
  if (first != null && last != null) throw new GraphQLError('Cannot use both first and last');
  if (after != null && before != null) throw new GraphQLError('Cannot use both after and before');
}

// Generic paginate function สำหรับ Prisma
export async function paginate<
  T extends { id: string; createdAt: Date },
  TFindManyArgs extends { where?: unknown; orderBy?: unknown; take?: number; skip?: number; cursor?: unknown }
>(
  findMany: (args: TFindManyArgs) => Promise<T[]>,
  count: (args: { where?: unknown }) => Promise<number>,
  args: PaginationArgs,
  baseWhere: unknown,
  orderBy: { createdAt: 'asc' | 'desc' } = { createdAt: 'desc' }
): Promise<PaginationResult<T>> {
  validatePaginationArgs(args);
  
  const { first, after, last, before } = args;
  const isForward = first != null || (!last && !before);
  const limit = (isForward ? first : last) ?? 20;
  
  let whereClause: Record<string, unknown> = { ...(baseWhere as object) };
  let take = limit + 1; // +1 เพื่อ detect hasNextPage
  let reverse = false;
  
  if (after) {
    const cursor = decodeCursor(after);
    whereClause.createdAt = { lt: new Date(cursor.value as string) };
  }
  
  if (before) {
    const cursor = decodeCursor(before);
    whereClause.createdAt = { gt: new Date(cursor.value as string) };
    reverse = true;
    take = -take;
  }
  
  // Fetch items
  const items = await findMany({
    where: whereClause,
    orderBy: isForward ? orderBy : { createdAt: orderBy.createdAt === 'asc' ? 'desc' : 'asc' },
    take: isForward ? take : -take,
  } as TFindManyArgs);
  
  const totalCount = await count({ where: baseWhere });
  
  // Detect has more
  let hasMore = items.length > limit;
  const nodes = hasMore ? items.slice(0, limit) : items;
  if (reverse) nodes.reverse();
  
  // Build edges
  const edges = nodes.map(node => ({
    node,
    cursor: encodeCursor({ id: node.id, value: node.createdAt.toISOString() }),
  }));
  
  return {
    edges,
    pageInfo: {
      hasNextPage: isForward ? hasMore : !!before,
      hasPreviousPage: isForward ? !!after : hasMore,
      startCursor: edges[0]?.cursor ?? null,
      endCursor: edges[edges.length - 1]?.cursor ?? null,
    },
    totalCount,
  };
}
```

---

## 📌 Step 914: Keyset Pagination สำหรับ Sort

```typescript
// Keyset pagination สำหรับ custom sort fields
// ดีกว่า offset pagination เพราะ stable และ efficient

export async function paginateWithSort<T extends { id: string }>(
  prisma: PrismaClient,
  model: 'product' | 'user' | 'post',
  args: PaginationArgs & {
    orderBy?: { field: string; direction: 'asc' | 'desc' };
    filter?: Record<string, unknown>;
  }
): Promise<PaginationResult<T>> {
  const { first = 20, after, orderBy = { field: 'createdAt', direction: 'desc' } } = args;
  
  let cursorCondition: Record<string, unknown> = {};
  
  if (after) {
    const cursor = decodeCursor(after);
    const dir = orderBy.direction;
    
    // Keyset: (sortValue, id) pair
    // ถ้า sort DESC: WHERE (sortField < cursorValue) OR (sortField = cursorValue AND id < cursorId)
    cursorCondition = {
      OR: [
        { [orderBy.field]: dir === 'desc' ? { lt: cursor.value } : { gt: cursor.value } },
        {
          [orderBy.field]: cursor.value,
          id: dir === 'desc' ? { lt: cursor.id } : { gt: cursor.id },
        },
      ],
    };
  }
  
  const client = prisma[model] as any;
  const items = await client.findMany({
    where: { ...args.filter, ...cursorCondition },
    orderBy: [
      { [orderBy.field]: orderBy.direction },
      { id: orderBy.direction }, // tiebreaker
    ],
    take: first + 1,
  });
  
  const hasNextPage = items.length > first;
  const nodes: T[] = hasNextPage ? items.slice(0, first) : items;
  
  const edges = nodes.map((node: T & Record<string, unknown>) => ({
    node,
    cursor: encodeCursor({
      id: node.id,
      value: node[orderBy.field] as string | number,
      field: orderBy.field,
    }),
  }));
  
  const totalCount = await client.count({ where: args.filter });
  
  return {
    edges,
    pageInfo: {
      hasNextPage,
      hasPreviousPage: !!after,
      startCursor: edges[0]?.cursor ?? null,
      endCursor: edges[edges.length - 1]?.cursor ?? null,
    },
    totalCount,
  };
}
```

---

## 📌 Step 915: Resolver Integration

```typescript
// src/resolvers/product.resolver.ts
import { paginate, paginateWithSort } from '../pagination/paginate.js';

const productResolvers = {
  Query: {
    products: async (_, args, ctx) => {
      const { first, after, last, before, filter, orderBy } = args;
      
      // Build filter
      const where: Record<string, unknown> = {};
      if (filter?.minPrice) where.price = { gte: filter.minPrice };
      if (filter?.maxPrice) where.price = { ...where.price, lte: filter.maxPrice };
      if (filter?.categoryId) where.categoryId = filter.categoryId;
      if (filter?.inStock) where.stock = { gt: 0 };
      if (filter?.search) {
        where.OR = [
          { name: { contains: filter.search, mode: 'insensitive' } },
          { description: { contains: filter.search, mode: 'insensitive' } },
        ];
      }
      
      // Map orderBy enum
      const orderByMap: Record<string, { field: string; direction: 'asc' | 'desc' }> = {
        CREATED_AT_DESC: { field: 'createdAt', direction: 'desc' },
        CREATED_AT_ASC: { field: 'createdAt', direction: 'asc' },
        PRICE_DESC: { field: 'price', direction: 'desc' },
        PRICE_ASC: { field: 'price', direction: 'asc' },
        NAME_ASC: { field: 'name', direction: 'asc' },
        NAME_DESC: { field: 'name', direction: 'desc' },
      };
      
      return paginateWithSort(ctx.prisma, 'product', {
        first,
        after,
        filter: where,
        orderBy: orderByMap[orderBy ?? 'CREATED_AT_DESC'],
      });
    },
  },
};
```

---

## 📌 Step 916: Client-Side Cursor Pagination

```typescript
// React + Apollo — infinite scroll
import { useProductsQuery } from '../generated/operations';

function ProductList() {
  const { data, loading, fetchMore } = useProductsQuery({
    variables: { first: 20 },
  });
  
  const loadMore = async () => {
    const endCursor = data?.products.pageInfo.endCursor;
    if (!endCursor) return;
    
    await fetchMore({
      variables: { first: 20, after: endCursor },
      // Apollo cache merge
      updateQuery: (prev, { fetchMoreResult }) => {
        if (!fetchMoreResult) return prev;
        return {
          products: {
            ...fetchMoreResult.products,
            edges: [
              ...prev.products.edges,
              ...fetchMoreResult.products.edges,
            ],
          },
        };
      },
    });
  };
  
  return (
    <div>
      {data?.products.edges.map(({ node, cursor }) => (
        <ProductCard key={cursor} product={node} />
      ))}
      
      {data?.products.pageInfo.hasNextPage && (
        <button onClick={loadMore} disabled={loading}>
          {loading ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}

// Apollo cache policy สำหรับ pagination
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        products: {
          keyArgs: ['filter', 'orderBy'], // cache key ไม่รวม pagination args
          merge(existing, incoming, { args }) {
            const merged = existing ? { ...existing } : { edges: [], pageInfo: {}, totalCount: 0 };
            
            if (args?.after) {
              // Append เมื่อมี after cursor
              merged.edges = [...merged.edges, ...incoming.edges];
            } else {
              // Replace เมื่อเริ่มใหม่
              merged.edges = incoming.edges;
            }
            
            merged.pageInfo = incoming.pageInfo;
            merged.totalCount = incoming.totalCount;
            return merged;
          },
        },
      },
    },
  },
});
```

---

## 📌 Step 917: สรุป Part 050

### เนื้อหาที่เรียนรู้

✅ Relay Connection Specification  
✅ Cursor encoding/decoding  
✅ Forward/backward pagination  
✅ Keyset pagination สำหรับ sort  
✅ Apollo cache integration  
✅ Total count strategies  

### Offset vs Cursor Pagination

```
Offset Pagination (skip/take):
✅ ง่ายต่อการ implement
✅ สามารถ jump ไปหน้าใดก็ได้
❌ Inconsistent เมื่อมีข้อมูลเพิ่ม/ลบ
❌ ช้าบน large datasets (OFFSET 10000 = scan 10000 rows)

Cursor Pagination (Relay):
✅ Stable — ข้อมูลเพิ่ม/ลบไม่ทำให้ตำแหน่งเปลี่ยน
✅ เร็วบน large datasets (index scan เท่านั้น)
✅ Relay spec — compatible กับ Relay client
❌ ไม่สามารถ jump ไปหน้าใดก็ได้
❌ ซับซ้อนกว่าสำหรับ multi-column sort
```

### ในส่วนถัดไป

➡️ **[Part 051](./part-051.md)** — Full-Text Search Integration

---

*Part 050 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
