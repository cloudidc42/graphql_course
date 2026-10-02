# Part 053 — Optimistic Updates & Apollo Cache 🚀

> **ระดับ:** Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1031–1070

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Apollo InMemoryCache architecture
- Cache normalization และ keyFields
- optimisticResponse ใน mutations
- Manual cache update ด้วย update function
- Cache eviction และ garbage collection
- Reactive variables
- Local-only fields
- Cache persistence (apollo3-cache-persist)

---

## 📌 Step 1031: Apollo Cache Architecture

```typescript
// Apollo InMemoryCache เก็บ data เป็น normalized objects
// ทุก object มี cache key = TypeName:id

// ตัวอย่าง normalized cache:
// {
//   "User:user-123": { id: "user-123", name: "Alice", __typename: "User" },
//   "Product:prod-456": { id: "prod-456", name: "Laptop", price: 999, __typename: "Product" },
//   "ROOT_QUERY": {
//     "me": { __ref: "User:user-123" },
//     "product({\"id\":\"prod-456\"})": { __ref: "Product:prod-456" }
//   }
// }

import { InMemoryCache, ApolloClient, makeVar, gql } from '@apollo/client';

const cache = new InMemoryCache({
  typePolicies: {
    // Custom keyFields
    User: {
      keyFields: ['id'],  // default
    },
    
    // Nested keyFields
    Address: {
      keyFields: ['street', 'city', 'country'],
    },
    
    // Singleton (no unique key)
    AppSettings: {
      keyFields: [],  // ไม่ normalize — singleton
    },
    
    Query: {
      fields: {
        // Pagination merge policy
        products: {
          keyArgs: ['filter', 'orderBy'], // cache ตาม args เหล่านี้เท่านั้น
          merge(existing = { edges: [] }, incoming, { args }) {
            if (!args?.after) return incoming;
            return {
              ...incoming,
              edges: [...existing.edges, ...incoming.edges],
            };
          },
        },
        
        // Redirect to existing cache entry
        product: {
          read(_, { args, toReference }) {
            return toReference({ __typename: 'Product', id: args?.id });
          },
        },
      },
    },
  },
});
```

---

## 📌 Step 1032: Optimistic Response

```typescript
// Optimistic UI: อัปเดต UI ก่อนที่ server จะ respond
// ถ้า server error → rollback อัตโนมัติ

import { useMutation, gql } from '@apollo/client';

const LIKE_POST = gql`
  mutation LikePost($postId: ID!) {
    likePost(postId: $postId) {
      id
      likesCount
      isLikedByMe
    }
  }
`;

function LikeButton({ post }: { post: { id: string; likesCount: number; isLikedByMe: boolean } }) {
  const [likePost] = useMutation(LIKE_POST, {
    // Optimistic response: ส่งทันที ก่อน server respond
    optimisticResponse: {
      likePost: {
        __typename: 'Post',
        id: post.id,
        likesCount: post.isLikedByMe ? post.likesCount - 1 : post.likesCount + 1,
        isLikedByMe: !post.isLikedByMe,
      },
    },
  });
  
  const handleLike = () => {
    likePost({ variables: { postId: post.id } });
  };
  
  return (
    <button onClick={handleLike}>
      {post.isLikedByMe ? '❤️' : '🤍'} {post.likesCount}
    </button>
  );
}

// สำหรับ create mutations — ต้องสร้าง temp ID
import { v4 as uuidv4 } from 'uuid';

const CREATE_COMMENT = gql`
  mutation CreateComment($postId: ID!, $content: String!) {
    createComment(postId: $postId, content: $content) {
      id
      content
      createdAt
      author { id name avatar }
    }
  }
`;

function CommentForm({ postId, currentUser }) {
  const [createComment] = useMutation(CREATE_COMMENT, {
    optimisticResponse: ({ content }) => ({
      createComment: {
        __typename: 'Comment',
        id: `temp-${uuidv4()}`,  // temp ID ที่จะถูก replace ด้วย real ID
        content,
        createdAt: new Date().toISOString(),
        author: {
          __typename: 'User',
          id: currentUser.id,
          name: currentUser.name,
          avatar: currentUser.avatar,
        },
      },
    }),
    // Update cache ด้วย comment ใหม่
    update(cache, { data }) {
      const newComment = data?.createComment;
      if (!newComment) return;
      
      cache.modify({
        id: cache.identify({ __typename: 'Post', id: postId }),
        fields: {
          comments(existingComments = []) {
            const newCommentRef = cache.writeFragment({
              data: newComment,
              fragment: gql`
                fragment NewComment on Comment {
                  id content createdAt
                  author { id name avatar }
                }
              `,
            });
            return [...existingComments, newCommentRef];
          },
          commentsCount(existing = 0) {
            return existing + 1;
          },
        },
      });
    },
  });
  
  // ...
}
```

---

## 📌 Step 1033: Manual Cache Updates

```typescript
// update function ใน useMutation — update cache manually

const DELETE_POST = gql`
  mutation DeletePost($id: ID!) {
    deletePost(id: $id) {
      id
    }
  }
`;

const [deletePost] = useMutation(DELETE_POST, {
  update(cache, { data }) {
    const deletedId = data?.deletePost.id;
    if (!deletedId) return;
    
    // Method 1: evict ออกจาก cache
    cache.evict({ id: cache.identify({ __typename: 'Post', id: deletedId }) });
    cache.gc(); // garbage collect dangling references
    
    // Method 2: modify list queries
    cache.modify({
      fields: {
        posts(existingPosts, { readField }) {
          return {
            ...existingPosts,
            edges: existingPosts.edges.filter(
              (edge: Reference) => readField('id', readField('node', edge)) !== deletedId
            ),
            totalCount: existingPosts.totalCount - 1,
          };
        },
      },
    });
  },
});

// refetchQueries: re-fetch หลัง mutation
const [updateCategory] = useMutation(UPDATE_CATEGORY, {
  // Re-fetch queries ที่อาจเปลี่ยน
  refetchQueries: [
    { query: GET_PRODUCTS, variables: { categoryId } },
    'GetCategories', // ชื่อ query
  ],
  // Block UI จนกว่า refetch เสร็จ
  awaitRefetchQueries: true,
});
```

---

## 📌 Step 1034: Reactive Variables

```typescript
// Reactive Variables: local state ที่ trigger re-render เมื่อเปลี่ยน

import { makeVar, useReactiveVar } from '@apollo/client';

// สร้าง reactive variables
export const isLoggedInVar = makeVar<boolean>(false);
export const cartItemsVar = makeVar<CartItem[]>([]);
export const themeVar = makeVar<'light' | 'dark'>('light');
export const notificationsVar = makeVar<Notification[]>([]);

// อ่าน/เขียน ใน components
function Header() {
  const isLoggedIn = useReactiveVar(isLoggedInVar);
  const cartItems = useReactiveVar(cartItemsVar);
  
  return (
    <header>
      {isLoggedIn ? <UserMenu /> : <LoginButton />}
      <CartIcon count={cartItems.length} />
    </header>
  );
}

// อ่าน/เขียน ใน resolvers
function loginUser(user: User) {
  isLoggedInVar(true);
  // เขียนข้อมูล user เข้า cache
  apolloClient.writeFragment({
    id: `User:${user.id}`,
    fragment: gql`fragment CurrentUser on User { id name email }`,
    data: user,
  });
}

// Cart operations
function addToCart(item: CartItem) {
  const current = cartItemsVar();
  const existing = current.find(i => i.productId === item.productId);
  
  if (existing) {
    cartItemsVar(current.map(i =>
      i.productId === item.productId
        ? { ...i, quantity: i.quantity + item.quantity }
        : i
    ));
  } else {
    cartItemsVar([...current, item]);
  }
}

// Reactive variable ใน cache policy
const cache = new InMemoryCache({
  typePolicies: {
    Product: {
      fields: {
        // Local-only field computed จาก reactive variable
        isInCart: {
          read(_, { readField }) {
            const productId = readField<string>('id');
            return cartItemsVar().some(item => item.productId === productId);
          },
        },
      },
    },
  },
});
```

---

## 📌 Step 1035: Local-Only Fields

```typescript
// Local fields: extend server types ด้วย client-only fields
// ไม่ส่ง field เหล่านี้ไปยัง server

const cache = new InMemoryCache({
  typePolicies: {
    Todo: {
      fields: {
        // Local field — ไม่มีใน server schema
        isSelected: {
          read(isSelected = false) {
            return isSelected;
          },
        },
      },
    },
    Product: {
      fields: {
        wishListed: {
          read(wishListed = false) {
            return wishListed;
          },
        },
      },
    },
  },
});

// Query ต้อง mark local fields ด้วย @client directive
const GET_TODOS = gql`
  query GetTodos {
    todos {
      id
      text
      completed
      isSelected @client  # local field
    }
  }
`;

const GET_PRODUCTS = gql`
  query GetProducts {
    products {
      edges {
        node {
          id
          name
          price
          wishListed @client  # local field
        }
      }
    }
  }
`;

// เขียน local field
function toggleSelect(todoId: string) {
  cache.modify({
    id: cache.identify({ __typename: 'Todo', id: todoId }),
    fields: {
      isSelected(prev) {
        return !prev;
      },
    },
  });
}
```

---

## 📌 Step 1036: Cache Persistence

```bash
npm install apollo3-cache-persist
```

```typescript
// src/apollo/persistedCache.ts
import { InMemoryCache } from '@apollo/client';
import { persistCache, LocalStorageWrapper } from 'apollo3-cache-persist';

export async function createPersistedCache() {
  const cache = new InMemoryCache({
    typePolicies: {
      // ... type policies
    },
  });
  
  await persistCache({
    cache,
    storage: new LocalStorageWrapper(window.localStorage),
    maxSize: 1048576,  // 1MB limit
    debug: process.env.NODE_ENV === 'development',
    
    // Custom serializer (optional)
    serialize: false, // ไม่ serialize — เร็วกว่า
  });
  
  return cache;
}

// สำหรับ React Native: ใช้ AsyncStorage
import AsyncStorage from '@react-native-async-storage/async-storage';

async function createRNCache() {
  const cache = new InMemoryCache();
  
  await persistCache({
    cache,
    storage: AsyncStorage,
    trigger: 'background', // persist เมื่อ app ไป background
  });
  
  return cache;
}

// App initialization
async function initApp() {
  const cache = await createPersistedCache();
  
  const client = new ApolloClient({
    uri: process.env.GRAPHQL_URL,
    cache,
    defaultOptions: {
      watchQuery: {
        // ใช้ cache ก่อน แล้ว fetch จาก network (stale-while-revalidate)
        fetchPolicy: 'cache-and-network',
      },
    },
  });
  
  return client;
}
```

---

## 📌 Step 1037: สรุป Part 053

### เนื้อหาที่เรียนรู้

✅ InMemoryCache normalization  
✅ Type policies และ keyFields  
✅ optimisticResponse  
✅ Manual cache updates  
✅ Reactive variables  
✅ Local-only fields (@client)  
✅ Cache persistence  

### Cache Strategies

```
fetchPolicy options:
- cache-first: ใช้ cache ก่อน (default) — เร็วที่สุด
- cache-and-network: แสดง cache ก่อน แล้ว re-fetch
- network-only: ดึงจาก network เสมอ — fresh data
- no-cache: ไม่ cache เลย
- cache-only: ดูแค่ cache เท่านั้น (offline)
- standby: ไม่ re-render อัตโนมัติ

nextFetchPolicy: policy หลังจาก first fetch
- 'cache-first': re-render จาก cache เท่านั้นหลัง initial fetch
```

### ในส่วนถัดไป

➡️ **[Part 054](./part-054.md)** — Server-Side Rendering with Next.js

---

*Part 053 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
