# Part 082 — GraphQL กับ Mobile: React Native & Expo 📱

> **ระดับ:** Expert | **เวลาเรียน:** 80 นาที | **ขั้นตอนที่:** 2166–2210

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Apollo Client ใน React Native
- Expo + GraphQL setup
- Offline-first patterns
- Optimistic UI สำหรับ mobile
- Push notifications กับ subscriptions
- Persisted cache ด้วย AsyncStorage
- Network detection & retry
- Mobile-specific performance tips

---

## 📌 Step 2166: Setup Apollo Client ใน React Native

```bash
# สร้าง Expo project
npx create-expo-app graphql-mobile --template expo-template-blank-typescript
cd graphql-mobile

# ติดตั้ง packages
npx expo install \
  @apollo/client \
  graphql \
  graphql-ws \
  @react-native-async-storage/async-storage \
  @react-native-community/netinfo
```

```typescript
// src/apollo/client.ts
import { ApolloClient, InMemoryCache, HttpLink, split, from } from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { createClient } from 'graphql-ws';
import { getMainDefinition } from '@apollo/client/utilities';
import { RetryLink } from '@apollo/client/link/retry';
import { onError } from '@apollo/client/link/error';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { AsyncStorageWrapper, persistCache } from 'apollo3-cache-persist';

// Persisted cache เพื่อ offline support
const cache = new InMemoryCache({
  typePolicies: {
    Product: { keyFields: ['id'] },
    User: { keyFields: ['id'] },
  },
});

// Initialize persisted cache (call ก่อน render)
export async function initApolloCache() {
  await persistCache({
    cache,
    storage: new AsyncStorageWrapper(AsyncStorage),
    maxSize: 1048576 * 5,  // 5MB
    debug: __DEV__,
  });
}

const httpLink = new HttpLink({
  uri: process.env.EXPO_PUBLIC_GRAPHQL_URL,
});

const wsLink = new GraphQLWsLink(
  createClient({
    url: process.env.EXPO_PUBLIC_WS_URL!,
    shouldRetry: () => true,
    retryAttempts: Infinity,
    retryWait: (n) => new Promise(r => setTimeout(r, Math.min(1000 * 2 ** n, 30000))),
  })
);

const errorLink = onError(({ graphQLErrors, networkError }) => {
  if (networkError) {
    // Mobile: network errors ปกติมาก (เข้าลิฟต์, ขึ้นรถไฟ)
    console.warn('Network error - will retry:', networkError.message);
  }
});

const retryLink = new RetryLink({
  delay: { initial: 500, max: 10000, jitter: true },
  attempts: { max: 5, retryIf: (err) => !!err },
});

const splitLink = split(
  ({ query }) => {
    const def = getMainDefinition(query);
    return def.kind === 'OperationDefinition' && def.operation === 'subscription';
  },
  wsLink,
  from([errorLink, retryLink, httpLink])
);

export const apolloClient = new ApolloClient({
  link: splitLink,
  cache,
  defaultOptions: {
    watchQuery: { fetchPolicy: 'cache-and-network' },
  },
});
```

---

## 📌 Step 2167: App Setup กับ Persisted Cache

```typescript
// App.tsx
import React, { useEffect, useState } from 'react';
import { View, ActivityIndicator } from 'react-native';
import { ApolloProvider } from '@apollo/client';
import { apolloClient, initApolloCache } from './src/apollo/client';
import RootNavigator from './src/navigation/RootNavigator';

export default function App() {
  const [cacheLoaded, setCacheLoaded] = useState(false);
  
  useEffect(() => {
    initApolloCache().then(() => setCacheLoaded(true));
  }, []);
  
  if (!cacheLoaded) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" />
      </View>
    );
  }
  
  return (
    <ApolloProvider client={apolloClient}>
      <RootNavigator />
    </ApolloProvider>
  );
}
```

---

## 📌 Step 2168: Network-Aware Fetching

```typescript
// src/hooks/useNetworkAwareQuery.ts
import { useQuery, DocumentNode } from '@apollo/client';
import NetInfo from '@react-native-community/netinfo';
import { useState, useEffect, useCallback } from 'react';

export function useNetworkAwareQuery<T>(
  query: DocumentNode,
  options: Parameters<typeof useQuery>[1] = {}
) {
  const [isOnline, setIsOnline] = useState(true);
  
  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state) => {
      setIsOnline(!!state.isConnected);
    });
    return unsubscribe;
  }, []);
  
  const result = useQuery<T>(query, {
    ...options,
    // เมื่อ offline: ใช้ cache เท่านั้น
    fetchPolicy: isOnline ? 'cache-and-network' : 'cache-only',
    // ไม่ retry ถ้า offline
    skip: options.skip,
  });
  
  const refetchWhenOnline = useCallback(() => {
    if (isOnline) result.refetch();
  }, [isOnline, result]);
  
  return {
    ...result,
    isOnline,
    refetchWhenOnline,
    // Convenient: มี cached data ถึงจะ offline
    hasData: !!result.data,
  };
}

// Usage
function ProductList() {
  const { data, loading, error, isOnline, hasData } = useNetworkAwareQuery<GetProductsQuery>(
    GET_PRODUCTS
  );
  
  return (
    <View>
      {!isOnline && (
        <Banner message="ออฟไลน์ — แสดงข้อมูลที่บันทึกไว้" type="warning" />
      )}
      {loading && !hasData && <LoadingSpinner />}
      {data?.products.map(p => <ProductCard key={p.id} product={p} />)}
    </View>
  );
}
```

---

## 📌 Step 2169: Optimistic UI สำหรับ Mobile

```typescript
// src/hooks/useAddToCart.ts
// Optimistic: UI update ทันที ก่อน server confirm
import { useMutation, gql } from '@apollo/client';

const ADD_TO_CART = gql`
  mutation AddToCart($productId: ID!, $quantity: Int!) {
    addToCart(productId: $productId, quantity: $quantity) {
      id
      items {
        product { id name price }
        quantity
      }
      totalAmount
    }
  }
`;

export function useAddToCart() {
  const [addToCart, { loading }] = useMutation(ADD_TO_CART, {
    optimisticResponse: ({ productId, quantity }) => ({
      __typename: 'Mutation',
      addToCart: {
        __typename: 'Cart',
        id: 'cart',
        items: [],        // จะ update จาก server
        totalAmount: 0,   // จะ recalculate
      },
    }),
    
    // Update cache จาก optimistic response
    update(cache, { data }) {
      // อัพเดต cart ใน cache
      cache.writeQuery({
        query: GET_CART,
        data: { cart: data?.addToCart },
      });
    },
    
    onError: (err) => {
      // Rollback optimistic update อัตโนมัติโดย Apollo
      Alert.alert('ไม่สามารถเพิ่มสินค้า', err.message);
    },
  });
  
  return {
    addToCart: (productId: string, quantity = 1) =>
      addToCart({ variables: { productId, quantity } }),
    loading,
  };
}
```

---

## 📌 Step 2170: Mobile Schema Patterns

```graphql
# schema/mobile.graphql
# BFF schema สำหรับ mobile: flat, efficient

type MobileProductFeed {
  items: [MobileProductCard!]!
  nextCursor: String
  hasMore: Boolean!
}

# Flattened สำหรับ mobile (ไม่ nested deep)
type MobileProductCard {
  id: ID!
  name: String!
  
  # Pre-formatted สำหรับ display
  priceDisplay: String!           # "฿1,250"
  discountDisplay: String         # "ลด 20%" หรือ null
  
  # Mobile-optimized image
  thumbnailUrl: String!           # 200x200
  
  # Pre-computed badge
  badge: MobileBadge
  
  isInCart: Boolean!
  isSaved: Boolean!
}

type MobileBadge {
  text: String!   # "ใหม่" / "ลดราคา" / "ขายดี"
  color: String!  # hex color
}

# Query เดียวได้ทุกอย่างที่หน้า home ต้องการ
type Query {
  homeFeed(cursor: String): MobileHomeFeed!
}

type MobileHomeFeed {
  banners: [Banner!]!
  featuredProducts: [MobileProductCard!]!
  categories: [Category!]!
  recommendedProducts: MobileProductFeed!
}
```

---

## 📌 Step 2171: Push Notifications + Subscriptions

```typescript
// src/notifications/setup.ts
import * as Notifications from 'expo-notifications';
import { apolloClient } from '../apollo/client';
import { gql } from '@apollo/client';

const REGISTER_PUSH_TOKEN = gql`
  mutation RegisterPushToken($token: String!, $platform: Platform!) {
    registerPushToken(token: $token, platform: $platform) {
      success
    }
  }
`;

export async function registerForPushNotifications() {
  const { status } = await Notifications.requestPermissionsAsync();
  if (status !== 'granted') return;
  
  const token = await Notifications.getExpoPushTokenAsync({
    projectId: process.env.EXPO_PUBLIC_PROJECT_ID,
  });
  
  // Register token กับ server
  await apolloClient.mutate({
    mutation: REGISTER_PUSH_TOKEN,
    variables: {
      token: token.data,
      platform: 'IOS', // หรือ ANDROID
    },
  });
}

// Server: ส่ง push notification เมื่อ order status เปลี่ยน
// แทนที่ WebSocket subscription สำหรับ background notifications
async function sendOrderStatusPushNotification(
  userId: string,
  orderId: string,
  status: OrderStatus
) {
  const pushToken = await getPushToken(userId);
  if (!pushToken) return;
  
  const messages = {
    CONFIRMED: { title: 'คำสั่งซื้อได้รับการยืนยัน', body: `คำสั่งซื้อ #${orderId} ได้รับการยืนยันแล้ว` },
    SHIPPED: { title: 'สินค้ากำลังจัดส่ง', body: `คำสั่งซื้อ #${orderId} กำลังจัดส่ง` },
    DELIVERED: { title: 'สินค้าจัดส่งแล้ว', body: `คำสั่งซื้อ #${orderId} จัดส่งเรียบร้อย` },
  };
  
  await fetch('https://exp.host/--/api/v2/push/send', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      to: pushToken,
      ...messages[status],
      data: { orderId, screen: 'OrderDetail' },
    }),
  });
}
```

---

## 📌 Step 2172: Mobile Performance Tips

```typescript
// 1. Fragment Colocation: component กำหนด data ที่ตัวเองต้องการ
const PRODUCT_CARD_FRAGMENT = gql`
  fragment ProductCardFields on Product {
    id
    name
    priceDisplay
    thumbnailUrl
    badge { text color }
    isInCart
  }
`;

function ProductCard({ product }: { product: ProductCardFieldsFragment }) {
  // Component ใช้แค่ fields ที่ fragment define
  return (
    <TouchableOpacity>
      <Image source={{ uri: product.thumbnailUrl }} style={styles.image} />
      <Text>{product.name}</Text>
      <Text>{product.priceDisplay}</Text>
      {product.badge && <Badge {...product.badge} />}
    </TouchableOpacity>
  );
}

// 2. FlashList แทน FlatList: ดีกว่ามากสำหรับ large lists
import { FlashList } from '@shopify/flash-list';

function ProductFeed() {
  const { data, fetchMore } = useQuery(GET_PRODUCTS_FEED);
  
  return (
    <FlashList
      data={data?.productFeed.items ?? []}
      renderItem={({ item }) => <ProductCard product={item} />}
      estimatedItemSize={120}  // ช่วย performance
      onEndReached={() => {
        if (data?.productFeed.hasMore) {
          fetchMore({
            variables: { cursor: data.productFeed.nextCursor },
            updateQuery: (prev, { fetchMoreResult }) => ({
              productFeed: {
                ...fetchMoreResult.productFeed,
                items: [...prev.productFeed.items, ...fetchMoreResult.productFeed.items],
              },
            }),
          });
        }
      }}
      onEndReachedThreshold={0.5}
    />
  );
}

// 3. Suspense สำหรับ lazy loading
import { Suspense } from 'react';

function ProductScreen() {
  return (
    <Suspense fallback={<ProductSkeleton />}>
      <ProductDetails />
    </Suspense>
  );
}
```

---

## 📌 Step 2173: สรุป Part 082

### เนื้อหาที่เรียนรู้

✅ Apollo Client + Expo setup  
✅ Persisted cache ด้วย AsyncStorage  
✅ Network-aware fetching  
✅ Optimistic UI  
✅ Push notifications  
✅ Mobile BFF schema  
✅ FlashList performance  

### Mobile GraphQL Checklist

```
Setup:
□ Persisted cache ด้วย AsyncStorage
□ Offline-first fetchPolicy
□ Network detection
□ Retry logic

Performance:
□ FlashList สำหรับ long lists
□ Fragment colocation
□ Pre-formatted fields (priceDisplay)
□ Thumbnail URLs แทน full images
□ Pagination ด้วย cursor

UX:
□ Optimistic updates
□ Skeleton loading
□ Offline banner
□ Push notifications สำหรับ background updates
```

### ในส่วนถัดไป

➡️ **[Part 083](./part-083.md)** — Advanced TypeScript กับ GraphQL

---

*Part 082 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
