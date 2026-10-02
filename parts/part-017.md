# Part 017 — Apollo Client & React Integration ⚛️

> **ระดับ:** Intermediate | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 551–590

---

## 🎯 สิ่งที่จะได้เรียนรู้

- ติดตั้ง Apollo Client
- InMemoryCache
- useQuery hook
- useMutation hook
- useSubscription hook
- useLazyQuery hook
- Cache policies
- Optimistic updates
- Apollo DevTools

---

## 📌 Step 551: ติดตั้ง Apollo Client

```bash
npm install @apollo/client graphql
```

### Setup Apollo Client

```javascript
// src/apollo/client.js
import {
  ApolloClient,
  InMemoryCache,
  HttpLink,
  ApolloLink,
  split,
} from '@apollo/client';
import { getMainDefinition } from '@apollo/client/utilities';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { createClient } from 'graphql-ws';
import { onError } from '@apollo/client/link/error';
import { setContext } from '@apollo/client/link/context';

// Error link
const errorLink = onError(({ graphQLErrors, networkError }) => {
  if (graphQLErrors) {
    for (const { message, extensions } of graphQLErrors) {
      if (extensions?.code === 'UNAUTHENTICATED') {
        // Clear token and redirect
        localStorage.removeItem('accessToken');
        window.location.href = '/login';
      }
    }
  }
});

// Auth link
const authLink = setContext((_, { headers }) => {
  const token = localStorage.getItem('accessToken');
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : '',
    },
  };
});

// HTTP link
const httpLink = new HttpLink({
  uri: import.meta.env.VITE_GRAPHQL_URL || 'http://localhost:4000/graphql',
});

// WebSocket link สำหรับ subscriptions
const wsLink = new GraphQLWsLink(
  createClient({
    url: import.meta.env.VITE_GRAPHQL_WS_URL || 'ws://localhost:4000/graphql',
    connectionParams: () => ({
      authorization: `Bearer ${localStorage.getItem('accessToken')}`,
    }),
  })
);

// Split: subscriptions → wsLink, queries/mutations → httpLink
const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    );
  },
  wsLink,
  ApolloLink.from([errorLink, authLink, httpLink])
);

// Cache configuration
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        // Merge pagination results
        users: {
          keyArgs: ['filter', 'sort'],
          merge(existing = { edges: [] }, incoming) {
            return {
              ...incoming,
              edges: [...existing.edges, ...incoming.edges],
            };
          },
        },
      },
    },
    User: {
      keyFields: ['id'],
    },
    Post: {
      keyFields: ['id'],
    },
  },
});

export const client = new ApolloClient({
  link: splitLink,
  cache,
  connectToDevTools: process.env.NODE_ENV === 'development',
});
```

### ApolloProvider

```javascript
// src/main.jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { ApolloProvider } from '@apollo/client';
import { client } from './apollo/client.js';
import App from './App.jsx';

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <ApolloProvider client={client}>
      <App />
    </ApolloProvider>
  </React.StrictMode>
);
```

---

## 📌 Step 552: useQuery Hook

```javascript
// src/components/UserList.jsx
import { useQuery, gql } from '@apollo/client';

const GET_USERS = gql`
  query GetUsers($limit: Int, $offset: Int) {
    users(limit: $limit, offset: $offset) {
      users {
        id
        name
        email
        avatar
        role
        createdAt
      }
      total
      hasMore
    }
  }
`;

export function UserList() {
  const { data, loading, error, refetch } = useQuery(GET_USERS, {
    variables: { limit: 10, offset: 0 },
    
    // Fetch policy
    fetchPolicy: 'cache-and-network',
    
    // Error policy
    errorPolicy: 'partial',
    
    // Skip query (conditional)
    skip: false,
    
    // Polling (auto refetch)
    // pollInterval: 5000,
    
    // Callbacks
    onCompleted: (data) => {
      console.log('Users loaded:', data.users.total);
    },
    onError: (error) => {
      console.error('Error loading users:', error);
    },
  });
  
  if (loading && !data) {
    return (
      <div className="grid grid-cols-3 gap-4">
        {[...Array(6)].map((_, i) => (
          <UserCardSkeleton key={i} />
        ))}
      </div>
    );
  }
  
  if (error) {
    return (
      <div className="error">
        <p>Error: {error.message}</p>
        <button onClick={() => refetch()}>Retry</button>
      </div>
    );
  }
  
  return (
    <div>
      <div className="grid grid-cols-3 gap-4">
        {data?.users.users.map(user => (
          <UserCard key={user.id} user={user} />
        ))}
      </div>
      
      {data?.users.hasMore && (
        <button onClick={() => refetch({ offset: data.users.users.length })}>
          Load More
        </button>
      )}
      
      <p>Total: {data?.users.total}</p>
    </div>
  );
}
```

---

## 📌 Step 553: useMutation Hook

```javascript
// src/components/CreateUserForm.jsx
import { useMutation, gql } from '@apollo/client';

const CREATE_USER = gql`
  mutation CreateUser($input: CreateUserInput!) {
    createUser(input: $input) {
      user {
        id
        name
        email
      }
      accessToken
    }
  }
`;

const GET_USERS = gql`query GetUsers { users { users { id name email } } }`;

export function CreateUserForm() {
  const [createUser, { loading, error, data }] = useMutation(CREATE_USER, {
    // Refetch queries หลัง mutation สำเร็จ
    refetchQueries: [GET_USERS],
    
    // หรือ update cache manually
    update(cache, { data: { createUser: { user } } }) {
      // อ่าน cache
      const existing = cache.readQuery({ query: GET_USERS });
      
      if (existing) {
        // เขียน cache ใหม่
        cache.writeQuery({
          query: GET_USERS,
          data: {
            users: {
              ...existing.users,
              users: [user, ...existing.users.users],
              total: existing.users.total + 1,
            },
          },
        });
      }
    },
    
    onCompleted: ({ createUser: { accessToken } }) => {
      localStorage.setItem('accessToken', accessToken);
      navigate('/dashboard');
    },
  });
  
  const [formData, setFormData] = useState({ name: '', email: '', password: '' });
  const [fieldErrors, setFieldErrors] = useState({});
  
  async function handleSubmit(e) {
    e.preventDefault();
    setFieldErrors({});
    
    try {
      await createUser({ variables: { input: formData } });
    } catch (err) {
      const gqlError = err.graphQLErrors?.[0];
      if (gqlError?.extensions?.code === 'VALIDATION_ERROR') {
        setFieldErrors(gqlError.extensions.errors);
      } else if (gqlError?.extensions?.code === 'CONFLICT') {
        setFieldErrors({ email: gqlError.message });
      }
    }
  }
  
  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <label>Name</label>
        <input
          value={formData.name}
          onChange={e => setFormData(p => ({ ...p, name: e.target.value }))}
          className={fieldErrors.name ? 'border-red-500' : ''}
        />
        {fieldErrors.name && <p className="text-red-500">{fieldErrors.name}</p>}
      </div>
      
      <div>
        <label>Email</label>
        <input
          type="email"
          value={formData.email}
          onChange={e => setFormData(p => ({ ...p, email: e.target.value }))}
          className={fieldErrors.email ? 'border-red-500' : ''}
        />
        {fieldErrors.email && <p className="text-red-500">{fieldErrors.email}</p>}
      </div>
      
      <div>
        <label>Password</label>
        <input
          type="password"
          value={formData.password}
          onChange={e => setFormData(p => ({ ...p, password: e.target.value }))}
          className={fieldErrors.password ? 'border-red-500' : ''}
        />
        {fieldErrors.password && <p className="text-red-500">{fieldErrors.password}</p>}
      </div>
      
      <button type="submit" disabled={loading}>
        {loading ? 'Creating...' : 'Create Account'}
      </button>
      
      {error && <p className="text-red-500">{error.message}</p>}
    </form>
  );
}
```

---

## 📌 Step 554: Optimistic Updates

```javascript
const [updateUser] = useMutation(UPDATE_USER, {
  optimisticResponse: ({ id, input }) => ({
    updateUser: {
      __typename: 'User',
      id,
      ...input,
    },
  }),
});

// ผล: UI อัพเดททันทีโดยไม่รอ server
// ถ้า server fail → rollback อัตโนมัติ
```

---

## 📌 Step 555: useSubscription Hook

```javascript
// src/components/Chat.jsx
import { useSubscription, gql } from '@apollo/client';

const NEW_MESSAGE = gql`
  subscription NewMessage($channelId: ID!) {
    newMessage(channelId: $channelId) {
      id
      content
      sender {
        id
        name
        avatar
      }
      createdAt
    }
  }
`;

export function Chat({ channelId }) {
  const [messages, setMessages] = useState([]);
  
  const { data, loading, error } = useSubscription(NEW_MESSAGE, {
    variables: { channelId },
    
    onData: ({ data: { data } }) => {
      if (data?.newMessage) {
        setMessages(prev => [...prev, data.newMessage]);
      }
    },
    
    onError: (error) => {
      console.error('Subscription error:', error);
    },
  });
  
  return (
    <div className="chat-container">
      {messages.map(msg => (
        <MessageBubble key={msg.id} message={msg} />
      ))}
    </div>
  );
}
```

---

## 📌 Step 556: Cache Policies

```javascript
// Fetch Policies
const { data } = useQuery(GET_USERS, {
  fetchPolicy: 'cache-first',        // ใช้ cache ก่อน, fetch ถ้าไม่มี (default)
  // fetchPolicy: 'network-only',    // fetch ทุกครั้ง, อัพเดท cache
  // fetchPolicy: 'cache-only',      // ใช้ cache เท่านั้น
  // fetchPolicy: 'no-cache',        // fetch ทุกครั้ง, ไม่ cache
  // fetchPolicy: 'cache-and-network', // ใช้ cache ทันที + fetch ใน background
  
  // nextFetchPolicy: ใช้สำหรับ subsequent renders
  nextFetchPolicy: 'cache-first',
});
```

---

## 📌 Step 557: สรุป Part 017

### เนื้อหาที่เรียนรู้

✅ Apollo Client setup + configuration  
✅ useQuery — load data  
✅ useMutation — modify data  
✅ useSubscription — real-time data  
✅ Optimistic updates  
✅ Cache management  
✅ Error handling  

### Homework

1. สร้าง React app ที่ใช้ Apollo Client
2. implement CRUD operations ด้วย useMutation
3. เพิ่ม optimistic UI สำหรับ like/unlike

### ในส่วนถัดไป

➡️ **[Part 018](./part-018.md)** — File Upload สำหรับ GraphQL

---

*Part 017 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
