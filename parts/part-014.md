# Part 014 — Pagination: Cursor & Offset 📄

> **ระดับ:** Intermediate | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 421–460

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Offset pagination
- Cursor-based pagination (Relay style)
- Connection pattern
- PageInfo type
- Forward/backward pagination
- keyset pagination
- Infinite scroll implementation
- GraphQL Cursor Connections Spec

---

## 📌 Step 421: Offset Pagination (แบบง่าย)

```graphql
type Query {
  users(
    limit: Int = 10
    offset: Int = 0
  ): UserListResult!
}

type UserListResult {
  users: [User!]!
  total: Int!
  hasMore: Boolean!
}
```

### Resolver

```javascript
const resolvers = {
  Query: {
    users: async (_, { limit = 10, offset = 0 }) => {
      const [users, total] = await Promise.all([
        User.findAll({ limit, offset, order: [['createdAt', 'DESC']] }),
        User.count(),
      ]);
      
      return {
        users,
        total,
        hasMore: offset + limit < total,
      };
    },
  },
};
```

### ปัญหาของ Offset Pagination

```
ปัญหา: ถ้ามีการเพิ่ม/ลบข้อมูลระหว่าง page
Page 1 (offset=0, limit=10): items 1-10
  → ลบ item 3 ออก
Page 2 (offset=10, limit=10): items 12-21 (แทนที่จะเป็น 11-20)
  → item 11 หายไป!
```

---

## 📌 Step 422: Cursor-based Pagination (Relay Spec)

```graphql
# Connection Types
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  node: User!
  cursor: String!    # opaque cursor สำหรับ item นี้
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String    # cursor ของ item แรก
  endCursor: String      # cursor ของ item สุดท้าย
}

type Query {
  users(
    first: Int          # จำนวน items ที่ต้องการ (forward pagination)
    after: String       # cursor ของ item สุดท้ายของหน้าก่อน
    last: Int           # จำนวน items (backward pagination)
    before: String      # cursor ของ item แรกของหน้าถัดไป
    filter: UserFilterInput
    sort: UserSortInput
  ): UserConnection!
}
```

---

## 📌 Step 423: Cursor Implementation

```javascript
// src/utils/cursor.js
import { Buffer } from 'buffer';

export function encodeCursor(data) {
  return Buffer.from(JSON.stringify(data)).toString('base64url');
}

export function decodeCursor(cursor) {
  try {
    return JSON.parse(Buffer.from(cursor, 'base64url').toString('utf8'));
  } catch {
    throw new GraphQLError('Invalid cursor', {
      extensions: { code: 'BAD_USER_INPUT' }
    });
  }
}

// สำหรับ time-based cursor (sorted by createdAt + id)
export function createCursor(item) {
  return encodeCursor({ id: item.id, createdAt: item.createdAt });
}
```

### Resolver สำหรับ Cursor Pagination

```javascript
const resolvers = {
  Query: {
    users: async (_, { first = 10, after, last, before, filter, sort }) => {
      // ป้องกันการขอข้อมูลมากเกินไป
      const limit = Math.min(first || last || 10, 100);
      
      let where = buildWhereClause(filter);
      let order = buildOrderClause(sort) || [['createdAt', 'DESC'], ['id', 'DESC']];
      
      // Decode cursor
      if (after) {
        const cursor = decodeCursor(after);
        where = {
          ...where,
          [Op.or]: [
            { createdAt: { [Op.lt]: cursor.createdAt } },
            {
              createdAt: cursor.createdAt,
              id: { [Op.lt]: cursor.id },
            },
          ],
        };
      }
      
      if (before) {
        const cursor = decodeCursor(before);
        where = {
          ...where,
          [Op.or]: [
            { createdAt: { [Op.gt]: cursor.createdAt } },
            {
              createdAt: cursor.createdAt,
              id: { [Op.gt]: cursor.id },
            },
          ],
        };
        order = [['createdAt', 'ASC'], ['id', 'ASC']];
      }
      
      // Fetch limit+1 เพื่อ check hasMore
      const users = await User.findAll({
        where,
        order,
        limit: limit + 1,
      });
      
      const hasMore = users.length > limit;
      const items = hasMore ? users.slice(0, limit) : users;
      
      if (before) items.reverse();
      
      // Build edges
      const edges = items.map(user => ({
        node: user,
        cursor: createCursor(user),
      }));
      
      return {
        edges,
        pageInfo: {
          hasNextPage: before ? true : hasMore,
          hasPreviousPage: after ? true : (before ? hasMore : false),
          startCursor: edges[0]?.cursor,
          endCursor: edges[edges.length - 1]?.cursor,
        },
        totalCount: User.count({ where: buildWhereClause(filter) }),
      };
    },
  },
};
```

---

## 📌 Step 424: Query ด้วย Cursor Pagination

```graphql
# Forward pagination — ไปหน้าถัดไป
query GetUsers($after: String) {
  users(first: 10, after: $after) {
    edges {
      node {
        id
        name
        email
        createdAt
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}
```

### Variables สำหรับหน้าแรก

```json
{ "after": null }
```

### Variables สำหรับหน้าถัดไป

```json
{ "after": "eyJpZCI6IjEwIiwiY3JlYXRlZEF0IjoiMjAyNC0wMS0wMSJ9" }
```

---

## 📌 Step 425: Infinite Scroll ด้วย Apollo Client

```javascript
// React + Apollo Client Infinite Scroll
import { useQuery, gql } from '@apollo/client';

const GET_USERS = gql`
  query GetUsers($after: String) {
    users(first: 10, after: $after) {
      edges {
        node {
          id
          name
          email
          avatar
        }
        cursor
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
`;

function UserList() {
  const { data, loading, error, fetchMore } = useQuery(GET_USERS, {
    variables: { after: null },
  });
  
  const loadMore = () => {
    if (!data?.users.pageInfo.hasNextPage) return;
    
    fetchMore({
      variables: {
        after: data.users.pageInfo.endCursor,
      },
      updateQuery: (prev, { fetchMoreResult }) => {
        if (!fetchMoreResult) return prev;
        
        return {
          users: {
            ...fetchMoreResult.users,
            edges: [
              ...prev.users.edges,
              ...fetchMoreResult.users.edges,
            ],
          },
        };
      },
    });
  };
  
  // Intersection Observer สำหรับ auto-load
  const sentinelRef = useRef(null);
  
  useEffect(() => {
    const observer = new IntersectionObserver((entries) => {
      if (entries[0].isIntersecting) {
        loadMore();
      }
    });
    
    if (sentinelRef.current) {
      observer.observe(sentinelRef.current);
    }
    
    return () => observer.disconnect();
  }, [data?.users.pageInfo.endCursor]);
  
  if (loading && !data) return <p>Loading...</p>;
  
  return (
    <div>
      {data?.users.edges.map(({ node }) => (
        <UserCard key={node.id} user={node} />
      ))}
      
      {loading && <p>Loading more...</p>}
      
      {/* Sentinel element สำหรับ IntersectionObserver */}
      <div ref={sentinelRef} />
    </div>
  );
}
```

---

## 📌 Step 426: Relay-style Connection Helper

```javascript
// src/utils/connection.js
export function buildConnection(items, { first, after, last, before }, totalCount) {
  const hasMore = items.length > (first || last);
  const nodes = hasMore ? items.slice(0, first || last) : items;
  
  if (before) nodes.reverse();
  
  const edges = nodes.map(node => ({
    node,
    cursor: createCursor(node),
  }));
  
  return {
    edges,
    nodes: edges.map(e => e.node),
    pageInfo: {
      hasNextPage: before ? true : hasMore,
      hasPreviousPage: after ? true : (before ? hasMore : false),
      startCursor: edges[0]?.cursor ?? null,
      endCursor: edges[edges.length - 1]?.cursor ?? null,
    },
    totalCount,
  };
}
```

---

## 📌 Step 427: สรุป Part 014

### เนื้อหาที่เรียนรู้

✅ Offset pagination — ง่ายแต่มีปัญหา  
✅ Cursor-based pagination (Relay spec)  
✅ Connection pattern (edges, node, cursor)  
✅ PageInfo type  
✅ Forward/backward pagination  
✅ Cursor encoding/decoding  
✅ Infinite scroll ด้วย Apollo Client  

### Homework

1. implement cursor pagination สำหรับ product list
2. สร้าง infinite scroll component สำหรับ product feed
3. เปรียบเทียบ offset vs cursor pagination performance

### ในส่วนถัดไป

➡️ **[Part 015](./part-015.md)** — DataLoader & N+1 Problem

---

*Part 014 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
