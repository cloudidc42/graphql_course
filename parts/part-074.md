# Part 074 — GraphQL with gRPC & Microservices 🔌

> **ระดับ:** Expert | **เวลาเรียน:** 85 นาที | **ขั้นตอนที่:** 1861–1900

---

## 🎯 สิ่งที่จะได้เรียนรู้

- gRPC + GraphQL integration
- Protocol translation patterns
- REST to GraphQL
- Message queue integration
- Service discovery
- GraphQL as API Gateway
- Distributed transactions
- GraphQL Mesh revisited

---

## 📌 Step 1861: gRPC Integration

```bash
npm install @grpc/grpc-js @grpc/proto-loader
```

```protobuf
// proto/users.proto
syntax = "proto3";
package users;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(CreateUserRequest) returns (User);
}

message GetUserRequest {
  string id = 1;
}

message User {
  string id = 1;
  string email = 2;
  string name = 3;
  string role = 4;
  int64 created_at = 5;
}

message ListUsersRequest {
  int32 page = 1;
  int32 limit = 2;
  string role_filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
}

message CreateUserRequest {
  string email = 1;
  string name = 2;
  string password = 3;
  string role = 4;
}
```

```typescript
// src/grpc/users-client.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import path from 'path';

const packageDefinition = protoLoader.loadSync(
  path.join(__dirname, '../proto/users.proto'),
  {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true,
  }
);

const proto = grpc.loadPackageDefinition(packageDefinition) as Record<string, unknown>;
const UsersProto = (proto.users as Record<string, unknown>).UserService as grpc.ServiceClientConstructor;

// gRPC client with retry & circuit breaker
export class UsersGrpcClient {
  private client: grpc.Client;
  
  constructor(private address: string = 'users-service:50051') {
    this.client = new UsersProto(
      address,
      grpc.credentials.createInsecure(),
      {
        'grpc.keepalive_time_ms': 10000,
        'grpc.keepalive_timeout_ms': 5000,
        'grpc.keepalive_permit_without_calls': 1,
        'grpc.max_reconnect_backoff_ms': 5000,
        'grpc.initial_reconnect_backoff_ms': 100,
      }
    );
  }
  
  // Promisify gRPC calls
  getUser(id: string): Promise<User> {
    return new Promise((resolve, reject) => {
      (this.client as Record<string, Function>).GetUser(
        { id },
        (err: grpc.ServiceError | null, response: User) => {
          if (err) {
            reject(translateGrpcError(err));
          } else {
            resolve(response);
          }
        }
      );
    });
  }
  
  listUsers(params: { page: number; limit: number; roleFilter?: string }): Promise<{ users: User[]; total: number }> {
    return new Promise((resolve, reject) => {
      (this.client as Record<string, Function>).ListUsers(
        { page: params.page, limit: params.limit, role_filter: params.roleFilter ?? '' },
        (err: grpc.ServiceError | null, response: { users: User[]; total: number }) => {
          if (err) reject(translateGrpcError(err));
          else resolve(response);
        }
      );
    });
  }
}

// แปลง gRPC errors → GraphQL errors
function translateGrpcError(err: grpc.ServiceError): GraphQLError {
  switch (err.code) {
    case grpc.status.NOT_FOUND:
      return new GraphQLError('Not found', { extensions: { code: 'NOT_FOUND' } });
    case grpc.status.PERMISSION_DENIED:
      return new GraphQLError('Forbidden', { extensions: { code: 'FORBIDDEN' } });
    case grpc.status.UNAUTHENTICATED:
      return new GraphQLError('Unauthenticated', { extensions: { code: 'UNAUTHENTICATED' } });
    case grpc.status.UNAVAILABLE:
      return new GraphQLError('Service unavailable', { extensions: { code: 'SERVICE_UNAVAILABLE' } });
    default:
      return new GraphQLError('Internal error', { extensions: { code: 'INTERNAL_ERROR' } });
  }
}
```

---

## 📌 Step 1862: GraphQL Resolver กับ gRPC

```typescript
// src/resolvers/users.resolver.ts

const usersGrpc = new UsersGrpcClient(process.env.USERS_GRPC_URL!);

export const userResolvers = {
  Query: {
    user: async (_, { id }, ctx) => {
      try {
        const grpcUser = await usersGrpc.getUser(id);
        return adaptGrpcUser(grpcUser);
      } catch (err) {
        if (err instanceof GraphQLError) throw err;
        throw new AppError('Failed to fetch user', ErrorCode.UPSTREAM_ERROR, { cause: err as Error });
      }
    },
    
    users: async (_, { first, after, filter }, ctx) => {
      const page = decodePage(after);
      const { users, total } = await usersGrpc.listUsers({
        page,
        limit: first ?? 20,
        roleFilter: filter?.role,
      });
      
      return {
        edges: users.map(u => ({
          node: adaptGrpcUser(u),
          cursor: encodeCursor({ page: page + 1 }),
        })),
        pageInfo: {
          hasNextPage: (page * (first ?? 20)) < total,
          hasPreviousPage: page > 1,
        },
        totalCount: total,
      };
    },
  },
};

// Adapt gRPC User → GraphQL User
function adaptGrpcUser(u: GrpcUser): GraphQLUser {
  return {
    id: u.id,
    email: u.email,
    name: u.name,
    role: u.role as UserRole,
    createdAt: new Date(parseInt(u.created_at) * 1000),
  };
}
```

---

## 📌 Step 1863: REST to GraphQL (REST Datasource)

```typescript
// src/datasources/payment-rest.ts
import { RESTDataSource } from '@apollo/datasource-rest';

class PaymentRESTDataSource extends RESTDataSource {
  override baseURL = process.env.PAYMENT_API_URL!;
  
  override willSendRequest(path: string, request: AugmentedRequest) {
    // Add auth headers
    request.headers['Authorization'] = `Bearer ${this.context.user?.token}`;
    request.headers['X-API-Version'] = '2024-01';
  }
  
  override async didReceiveResponse<T>(
    response: Response,
    request: Request
  ): Promise<T> {
    if (!response.ok) {
      const body = await response.json() as { message?: string; code?: string };
      throw new GraphQLError(
        body.message ?? 'Payment service error',
        { extensions: { code: body.code ?? 'PAYMENT_ERROR', httpStatus: response.status } }
      );
    }
    return super.didReceiveResponse(response, request);
  }
  
  // REST methods → GraphQL resolvers
  async getPaymentMethod(id: string) {
    return this.get<PaymentMethod>(`/payment-methods/${id}`);
  }
  
  async listPaymentMethods(userId: string) {
    return this.get<PaymentMethod[]>(`/users/${userId}/payment-methods`);
  }
  
  async processPayment(input: ProcessPaymentInput) {
    return this.post<PaymentResult>('/charges', { body: input });
  }
  
  async refundPayment(chargeId: string, amount?: number) {
    return this.post<RefundResult>(`/charges/${chargeId}/refunds`, {
      body: { amount },
    });
  }
}

// ใช้ใน context
const createContext = ({ req }): AppContext => ({
  user: getUser(req),
  dataSources: {
    payment: new PaymentRESTDataSource(),
  },
});
```

---

## 📌 Step 1864: Saga Pattern สำหรับ Distributed Transactions

```typescript
// src/sagas/checkout.saga.ts
// Saga: orchestrated distributed transaction

interface CheckoutSagaState {
  orderId?: string;
  paymentId?: string;
  inventoryReserved?: boolean;
  paymentCharged?: boolean;
}

export class CheckoutSaga {
  private state: CheckoutSagaState = {};
  
  async execute(input: CheckoutInput, ctx: AppContext): Promise<Order> {
    try {
      // Step 1: Reserve inventory
      await this.reserveInventory(input);
      
      // Step 2: Create order
      const order = await this.createOrder(input, ctx);
      
      // Step 3: Charge payment
      await this.chargePayment(order, input);
      
      // Step 4: Confirm order
      await this.confirmOrder(order.id, ctx);
      
      return order;
      
    } catch (err) {
      // Compensating transactions (rollback)
      await this.compensate(ctx);
      throw err;
    }
  }
  
  private async reserveInventory(input: CheckoutInput) {
    const result = await inventoryService.reserve({
      items: input.items.map(i => ({ productId: i.productId, quantity: i.quantity })),
      ttl: 900,  // 15 minutes
    });
    
    this.state.inventoryReserved = true;
    return result;
  }
  
  private async createOrder(input: CheckoutInput, ctx: AppContext) {
    const order = await ctx.prisma.order.create({
      data: {
        customerId: ctx.user!.id,
        status: 'PENDING',
        items: { create: input.items },
        total: calculateTotal(input.items),
      },
    });
    
    this.state.orderId = order.id;
    return order;
  }
  
  private async chargePayment(order: Order, input: CheckoutInput) {
    const payment = await paymentService.charge({
      amount: order.total,
      currency: 'THB',
      paymentMethodId: input.paymentMethodId,
      orderId: order.id,
    });
    
    this.state.paymentId = payment.id;
    this.state.paymentCharged = true;
    return payment;
  }
  
  // Compensating transactions
  private async compensate(ctx: AppContext) {
    const errors: Error[] = [];
    
    // Reverse in opposite order
    if (this.state.paymentCharged && this.state.paymentId) {
      try {
        await paymentService.refund(this.state.paymentId);
      } catch (e) {
        errors.push(e as Error);
      }
    }
    
    if (this.state.orderId) {
      try {
        await ctx.prisma.order.update({
          where: { id: this.state.orderId },
          data: { status: 'CANCELLED' },
        });
      } catch (e) {
        errors.push(e as Error);
      }
    }
    
    if (this.state.inventoryReserved) {
      try {
        await inventoryService.releaseReservation(this.state.orderId!);
      } catch (e) {
        errors.push(e as Error);
      }
    }
    
    if (errors.length) {
      console.error('Compensation errors:', errors);
    }
  }
}
```

---

## 📌 Step 1865: สรุป Part 074

### เนื้อหาที่เรียนรู้

✅ gRPC client สำหรับ GraphQL  
✅ Error translation (gRPC → GraphQL)  
✅ REST DataSource  
✅ Saga pattern  
✅ Distributed transaction compensation  

### Integration Patterns Summary

```
Use gRPC when:
- High-performance internal services
- Strongly-typed contracts between services
- Streaming data between services

Use REST DataSource when:
- Integrating 3rd party APIs
- Legacy systems
- Simple request/response

Use Message Queue when:
- Async operations (email, notifications)
- Event-driven updates
- Decoupled services

Use Saga when:
- Multi-step transactions across services
- Need rollback on failure
```

### ในส่วนถัดไป

➡️ **[Part 075](./part-075.md)** — Advanced Subscriptions & Real-Time Patterns

---

*Part 074 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
