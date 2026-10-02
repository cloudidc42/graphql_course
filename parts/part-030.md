# Part 030 — Real-world GraphQL API: E-commerce 🛒

> **ระดับ:** Advanced | **เวลาเรียน:** 240 นาที | **ขั้นตอนที่:** 1081–1150

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Complete e-commerce GraphQL API
- Product catalog, search, filtering
- Shopping cart
- Order management
- Payment integration
- Reviews & ratings
- Inventory management
- Admin dashboard
- Real-time order tracking

---

## 📌 Step 1081: Project Structure

```
ecommerce-graphql/
├── src/
│   ├── schema/
│   │   ├── user.graphql
│   │   ├── product.graphql
│   │   ├── order.graphql
│   │   ├── cart.graphql
│   │   ├── review.graphql
│   │   └── index.graphql
│   ├── resolvers/
│   │   ├── user.resolver.js
│   │   ├── product.resolver.js
│   │   ├── order.resolver.js
│   │   ├── cart.resolver.js
│   │   └── index.js
│   ├── models/
│   ├── services/
│   │   ├── payment.service.js
│   │   ├── email.service.js
│   │   ├── search.service.js
│   │   └── inventory.service.js
│   ├── loaders/
│   │   └── index.js
│   ├── middleware/
│   └── index.js
├── prisma/
│   └── schema.prisma
├── docker-compose.yml
└── package.json
```

---

## 📌 Step 1082: Complete Schema

```graphql
# schema/product.graphql
scalar Decimal
scalar DateTime
scalar URL

type Product {
  id: ID!
  name: String!
  slug: String!
  description: String
  shortDescription: String
  sku: String!
  
  price: Decimal!
  compareAtPrice: Decimal
  costPerItem: Decimal
  
  category: Category!
  brand: Brand
  seller: User!
  
  images: [ProductImage!]!
  primaryImage: ProductImage
  
  variants: [ProductVariant!]!
  options: [ProductOption!]!
  
  inventory: ProductInventory!
  
  status: ProductStatus!
  isDigital: Boolean!
  
  rating: Float
  reviewCount: Int!
  reviews(first: Int, after: String): ReviewConnection!
  
  tags: [Tag!]!
  
  relatedProducts: [Product!]!
  
  createdAt: DateTime!
  updatedAt: DateTime!
  
  # Computed
  inStock: Boolean!
  discount: DiscountInfo
  seoMeta: SEOMeta
}

type ProductVariant {
  id: ID!
  product: Product!
  title: String!
  sku: String!
  price: Decimal!
  compareAtPrice: Decimal
  inventory: ProductInventory!
  options: [VariantOption!]!
  image: ProductImage
}

type ProductInventory {
  quantity: Int!
  reservedQuantity: Int!
  availableQuantity: Int!
  trackQuantity: Boolean!
  allowBackorders: Boolean!
}

type DiscountInfo {
  amount: Decimal!
  percentage: Float!
  type: DiscountType!
  expiresAt: DateTime
}

enum ProductStatus { ACTIVE DRAFT ARCHIVED }
enum DiscountType { PERCENTAGE FIXED_AMOUNT }

# schema/cart.graphql
type Cart {
  id: ID!
  customer: User
  sessionId: String
  items: [CartItem!]!
  
  subtotal: Decimal!
  discountAmount: Decimal!
  shippingEstimate: Decimal
  taxAmount: Decimal!
  total: Decimal!
  
  coupon: Coupon
  
  itemCount: Int!
  isValid: Boolean!
  warnings: [String!]!
  
  updatedAt: DateTime!
  expiresAt: DateTime!
}

type CartItem {
  id: ID!
  product: Product!
  variant: ProductVariant
  quantity: Int!
  price: Decimal!
  total: Decimal!
  
  isAvailable: Boolean!
  availableQuantity: Int!
}

# schema/order.graphql
type Order {
  id: ID!
  orderNumber: String!
  customer: User!
  
  status: OrderStatus!
  paymentStatus: PaymentStatus!
  fulfillmentStatus: FulfillmentStatus!
  
  items: [OrderItem!]!
  
  subtotal: Decimal!
  discountAmount: Decimal!
  shippingAmount: Decimal!
  taxAmount: Decimal!
  total: Decimal!
  
  shippingAddress: Address!
  billingAddress: Address!
  
  payment: Payment
  shipments: [Shipment!]!
  
  notes: String
  
  createdAt: DateTime!
  updatedAt: DateTime!
  
  # Real-time tracking
  tracking: OrderTracking
}

enum OrderStatus { PENDING CONFIRMED PROCESSING SHIPPED DELIVERED CANCELLED REFUNDED }
enum PaymentStatus { PENDING PAID FAILED REFUNDED PARTIALLY_REFUNDED }
enum FulfillmentStatus { UNFULFILLED PARTIALLY_FULFILLED FULFILLED }
```

---

## 📌 Step 1083: Cart Resolvers

```javascript
// src/resolvers/cart.resolver.js

const CART_TTL = 60 * 60 * 24 * 7; // 7 days

export const cartResolvers = {
  Query: {
    cart: async (_, { cartId }, { prisma, user, redis }) => {
      const id = cartId || (user ? `user_${user.sub}` : null);
      if (!id) return null;
      
      // Try cache first
      const cached = await redis.get(`cart:${id}`);
      if (cached) return JSON.parse(cached);
      
      // Get from DB
      const cart = await prisma.cart.findUnique({
        where: user ? { customerId: user.sub } : { id: cartId },
        include: {
          items: {
            include: {
              product: { include: { images: true } },
              variant: true,
            },
          },
          coupon: true,
        },
      });
      
      return cart ? enrichCart(cart) : createEmptyCart(id);
    },
  },
  
  Mutation: {
    addToCart: async (_, { input }, { prisma, user, redis }) => {
      const { productId, variantId, quantity } = input;
      const cartId = user ? `user_${user.sub}` : input.cartId;
      
      // Validate product
      const product = await prisma.product.findUnique({
        where: { id: productId },
        include: { inventory: true },
      });
      
      if (!product || product.status !== 'ACTIVE') {
        throw new GraphQLError('Product not available');
      }
      
      const availableQty = product.inventory.availableQuantity;
      if (!product.inventory.allowBackorders && quantity > availableQty) {
        throw new GraphQLError(`Only ${availableQty} items available`);
      }
      
      // Upsert cart item
      const cart = await prisma.cart.upsert({
        where: user ? { customerId: user.sub } : { id: cartId },
        create: {
          id: cartId,
          customerId: user?.sub,
          items: {
            create: [{
              productId,
              variantId,
              quantity,
              price: product.price,
            }],
          },
        },
        update: {
          items: {
            upsert: {
              where: { cartId_productId_variantId: { cartId, productId, variantId: variantId || '' } },
              create: { productId, variantId, quantity, price: product.price },
              update: { quantity: { increment: quantity } },
            },
          },
        },
        include: {
          items: { include: { product: true, variant: true } },
          coupon: true,
        },
      });
      
      // Invalidate cache
      await redis.del(`cart:${cartId}`);
      
      return enrichCart(cart);
    },
    
    removeFromCart: async (_, { cartItemId }, { prisma, user }) => {
      const item = await prisma.cartItem.delete({ where: { id: cartItemId } });
      return prisma.cart.findUnique({
        where: { id: item.cartId },
        include: { items: { include: { product: true } }, coupon: true },
      });
    },
    
    applyCartCoupon: async (_, { cartId, code }, { prisma }) => {
      const coupon = await prisma.coupon.findUnique({
        where: { code: code.toUpperCase() },
      });
      
      if (!coupon || !coupon.isActive) {
        throw new GraphQLError('Invalid coupon code');
      }
      
      if (coupon.expiresAt && coupon.expiresAt < new Date()) {
        throw new GraphQLError('Coupon has expired');
      }
      
      if (coupon.usageCount >= coupon.maxUsage) {
        throw new GraphQLError('Coupon usage limit reached');
      }
      
      return prisma.cart.update({
        where: { id: cartId },
        data: { couponId: coupon.id },
        include: { items: { include: { product: true } }, coupon: true },
      });
    },
    
    checkoutCart: async (_, { cartId, input }, { prisma, user }) => {
      if (!user) throw new GraphQLError('Must be logged in to checkout');
      
      return prisma.$transaction(async (tx) => {
        // Get cart
        const cart = await tx.cart.findUnique({
          where: { id: cartId },
          include: {
            items: { include: { product: { include: { inventory: true } } } },
            coupon: true,
          },
        });
        
        if (!cart || cart.items.length === 0) {
          throw new GraphQLError('Cart is empty');
        }
        
        // Reserve inventory
        for (const item of cart.items) {
          const result = await tx.productInventory.updateMany({
            where: {
              productId: item.productId,
              availableQuantity: { gte: item.quantity },
            },
            data: {
              reservedQuantity: { increment: item.quantity },
            },
          });
          
          if (result.count === 0) {
            throw new GraphQLError(`${item.product.name} is out of stock`);
          }
        }
        
        // Calculate totals
        const subtotal = cart.items.reduce(
          (sum, item) => sum + item.price * item.quantity, 0
        );
        const discountAmount = calculateDiscount(cart.coupon, subtotal);
        const shippingAmount = calculateShipping(input.shippingAddress);
        const taxAmount = (subtotal - discountAmount) * 0.07;
        const total = subtotal - discountAmount + shippingAmount + taxAmount;
        
        // Create order
        const order = await tx.order.create({
          data: {
            orderNumber: generateOrderNumber(),
            customerId: user.sub,
            status: 'PENDING',
            paymentStatus: 'PENDING',
            subtotal,
            discountAmount,
            shippingAmount,
            taxAmount,
            total,
            shippingAddress: { create: input.shippingAddress },
            billingAddress: { create: input.billingAddress || input.shippingAddress },
            items: {
              create: cart.items.map(item => ({
                productId: item.productId,
                variantId: item.variantId,
                quantity: item.quantity,
                price: item.price,
                total: item.price * item.quantity,
              })),
            },
          },
          include: { items: { include: { product: true } } },
        });
        
        // Clear cart
        await tx.cart.delete({ where: { id: cartId } });
        
        // Publish event
        await pubsub.publish('ORDER_CREATED', { orderCreated: order });
        
        return order;
      });
    },
  },
};

function enrichCart(cart) {
  const subtotal = cart.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const discountAmount = cart.coupon ? calculateDiscount(cart.coupon, subtotal) : 0;
  const taxAmount = (subtotal - discountAmount) * 0.07;
  
  return {
    ...cart,
    subtotal,
    discountAmount,
    taxAmount,
    total: subtotal - discountAmount + taxAmount,
    itemCount: cart.items.reduce((sum, item) => sum + item.quantity, 0),
  };
}
```

---

## 📌 Step 1084: Real-time Order Tracking

```graphql
type OrderTracking {
  orderId: ID!
  status: OrderStatus!
  updates: [TrackingUpdate!]!
  currentLocation: Location
  estimatedDelivery: DateTime
}

type TrackingUpdate {
  id: ID!
  status: String!
  description: String!
  location: String
  timestamp: DateTime!
}

type Subscription {
  orderStatusUpdated(orderId: ID!): Order!
}
```

```javascript
const resolvers = {
  Subscription: {
    orderStatusUpdated: {
      subscribe: withFilter(
        () => pubsub.asyncIterator('ORDER_STATUS_UPDATED'),
        async (payload, { orderId }, { user }) => {
          if (!user) return false;
          
          // Only notify order owner
          const order = await Order.findById(orderId);
          return order?.customerId === user.sub &&
            payload.orderStatusUpdated.id === orderId;
        }
      ),
    },
  },
  
  Mutation: {
    updateOrderStatus: async (_, { orderId, status, note }, { user }) => {
      if (user.role !== 'ADMIN') throw new GraphQLError('Forbidden');
      
      const order = await prisma.order.update({
        where: { id: orderId },
        data: {
          status,
          trackingUpdates: {
            create: {
              status,
              description: note || `Order ${status.toLowerCase()}`,
            },
          },
        },
      });
      
      // Publish real-time update
      await pubsub.publish('ORDER_STATUS_UPDATED', {
        orderStatusUpdated: order,
      });
      
      // Send email notification
      await emailQueue.add('order-status', {
        orderId: order.id,
        status,
        customerEmail: order.customer.email,
      });
      
      return order;
    },
  },
};
```

---

## 📌 Step 1085: สรุป Part 030

### เนื้อหาที่เรียนรู้

✅ Complete e-commerce schema  
✅ Cart management  
✅ Checkout flow ด้วย transactions  
✅ Inventory reservation  
✅ Coupon system  
✅ Real-time order tracking  
✅ Event publishing  

### Homework

1. implement complete e-commerce API
2. เพิ่ม payment integration (Stripe, Omise)
3. สร้าง admin dashboard mutations

### ในส่วนถัดไป

➡️ **[Part 031](./part-031.md)** — GraphQL Security: Rate Limiting & DoS Prevention

---

*Part 030 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
