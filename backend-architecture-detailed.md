# Backend Architecture Deep Dive

## Overview

The backend architecture follows a **Domain-Driven Design (DDD)** approach with **microservices**, emphasizing clear boundaries, event-driven communication, and database-per-service patterns. Each service is independently deployable, scalable, and owns its data.

## Service Architecture Layers

### 1. **Core Platform Services**

These services handle cross-cutting concerns and foundational functionality:

#### **User Service**
```typescript
// User Service API Structure
interface UserService {
  // User Management
  createUser(userData: CreateUserRequest): Promise<User>
  getUserById(userId: string): Promise<User>
  updateUser(userId: string, updates: UpdateUserRequest): Promise<User>
  deleteUser(userId: string): Promise<void>
  
  // Profile Management
  updateProfile(userId: string, profile: ProfileData): Promise<UserProfile>
  getUserPreferences(userId: string): Promise<UserPreferences>
  
  // Events Published
  // - UserCreated, UserUpdated, UserDeleted, ProfileUpdated
}

// Data Model
interface User {
  id: string
  email: string
  username: string
  status: UserStatus
  createdAt: Date
  updatedAt: Date
  tenantId: string
}
```

#### **Authentication Service**
```typescript
interface AuthService {
  // Authentication
  login(credentials: LoginRequest): Promise<AuthResponse>
  logout(token: string): Promise<void>
  refreshToken(refreshToken: string): Promise<AuthResponse>
  
  // Authorization
  validateToken(token: string): Promise<TokenValidation>
  checkPermissions(userId: string, resource: string, action: string): Promise<boolean>
  
  // Multi-factor Authentication
  enableMFA(userId: string, method: MFAMethod): Promise<MFASetup>
  verifyMFA(userId: string, code: string): Promise<boolean>
  
  // OAuth/SSO
  initiateOAuth(provider: string): Promise<OAuthUrl>
  handleOAuthCallback(provider: string, code: string): Promise<AuthResponse>
}

// JWT Token Structure
interface JWTPayload {
  sub: string        // User ID
  email: string
  tenantId: string
  roles: string[]
  permissions: string[]
  exp: number
  iat: number
}
```

#### **Tenant Service (Multi-tenancy)**
```typescript
interface TenantService {
  // Tenant Management
  createTenant(tenantData: CreateTenantRequest): Promise<Tenant>
  getTenant(tenantId: string): Promise<Tenant>
  updateTenant(tenantId: string, updates: UpdateTenantRequest): Promise<Tenant>
  
  // Subscription & Features
  getTenantFeatures(tenantId: string): Promise<TenantFeatures>
  updateSubscription(tenantId: string, plan: SubscriptionPlan): Promise<void>
  
  // User-Tenant Relationships
  addUserToTenant(userId: string, tenantId: string, role: string): Promise<void>
  removeUserFromTenant(userId: string, tenantId: string): Promise<void>
  getUserTenants(userId: string): Promise<Tenant[]>
}

interface Tenant {
  id: string
  name: string
  domain: string
  subscriptionPlan: SubscriptionPlan
  settings: TenantSettings
  features: string[]
  status: TenantStatus
}
```

#### **Billing Service**
```typescript
interface BillingService {
  // Subscription Management
  createSubscription(tenantId: string, plan: SubscriptionPlan): Promise<Subscription>
  updateSubscription(subscriptionId: string, plan: SubscriptionPlan): Promise<Subscription>
  cancelSubscription(subscriptionId: string): Promise<void>
  
  // Payment Processing
  processPayment(tenantId: string, amount: number, method: PaymentMethod): Promise<Payment>
  addPaymentMethod(tenantId: string, paymentMethod: PaymentMethod): Promise<void>
  
  // Usage Tracking
  recordUsage(tenantId: string, feature: string, quantity: number): Promise<void>
  getUsageReport(tenantId: string, period: DateRange): Promise<UsageReport>
  
  // Invoicing
  generateInvoice(tenantId: string, period: DateRange): Promise<Invoice>
  getInvoices(tenantId: string): Promise<Invoice[]>
}
```

### 2. **Domain Services**

These are business-specific services that implement core domain logic:

#### **Domain Service Pattern**
```typescript
// Example: E-commerce Domain Service
interface ProductService {
  // Product Management
  createProduct(product: CreateProductRequest): Promise<Product>
  updateProduct(productId: string, updates: UpdateProductRequest): Promise<Product>
  getProduct(productId: string): Promise<Product>
  searchProducts(query: ProductSearchQuery): Promise<ProductSearchResult>
  
  // Inventory
  updateInventory(productId: string, quantity: number): Promise<void>
  checkAvailability(productId: string, quantity: number): Promise<boolean>
  
  // Events Published
  // - ProductCreated, ProductUpdated, InventoryUpdated, ProductOutOfStock
}

interface OrderService {
  // Order Management
  createOrder(order: CreateOrderRequest): Promise<Order>
  updateOrderStatus(orderId: string, status: OrderStatus): Promise<Order>
  getOrder(orderId: string): Promise<Order>
  getUserOrders(userId: string): Promise<Order[]>
  
  // Order Processing
  processPayment(orderId: string): Promise<PaymentResult>
  fulfillOrder(orderId: string): Promise<void>
  cancelOrder(orderId: string, reason: string): Promise<void>
  
  // Events Published
  // - OrderCreated, OrderPaid, OrderShipped, OrderCancelled
}
```

### 3. **Platform Services**

Supporting services that provide common functionality:

#### **Notification Service**
```typescript
interface NotificationService {
  // Multi-channel Notifications
  sendEmail(to: string, template: string, data: any): Promise<void>
  sendSMS(to: string, message: string): Promise<void>
  sendPushNotification(userId: string, message: PushMessage): Promise<void>
  sendInAppNotification(userId: string, notification: InAppNotification): Promise<void>
  
  // Template Management
  createTemplate(template: NotificationTemplate): Promise<void>
  updateTemplate(templateId: string, template: NotificationTemplate): Promise<void>
  
  // Preferences
  updateUserPreferences(userId: string, preferences: NotificationPreferences): Promise<void>
  
  // Bulk Operations
  sendBulkEmail(recipients: string[], template: string, data: any): Promise<void>
}
```

#### **File Service**
```typescript
interface FileService {
  // File Operations
  uploadFile(file: FileUploadRequest): Promise<FileMetadata>
  downloadFile(fileId: string): Promise<FileDownloadResponse>
  deleteFile(fileId: string): Promise<void>
  
  // Image Processing
  resizeImage(fileId: string, dimensions: ImageDimensions): Promise<FileMetadata>
  generateThumbnail(fileId: string): Promise<FileMetadata>
  
  // Access Control
  generatePresignedUrl(fileId: string, expiration: number): Promise<string>
  setFilePermissions(fileId: string, permissions: FilePermissions): Promise<void>
  
  // Metadata
  getFileMetadata(fileId: string): Promise<FileMetadata>
  updateFileMetadata(fileId: string, metadata: FileMetadata): Promise<void>
}
```

## Service Communication Patterns

### 1. **Synchronous Communication**

#### **REST APIs**
```typescript
// Standard REST API structure for each service
class ProductController {
  @Get('/products/:id')
  async getProduct(@Param('id') id: string): Promise<ProductResponse> {
    return await this.productService.getProduct(id);
  }
  
  @Post('/products')
  @Auth('product:create')
  async createProduct(@Body() product: CreateProductRequest): Promise<ProductResponse> {
    return await this.productService.createProduct(product);
  }
  
  @Put('/products/:id')
  @Auth('product:update')
  async updateProduct(
    @Param('id') id: string, 
    @Body() updates: UpdateProductRequest
  ): Promise<ProductResponse> {
    return await this.productService.updateProduct(id, updates);
  }
}
```

#### **GraphQL Federation**
```typescript
// GraphQL schema for unified API
type Query {
  user(id: ID!): User
  product(id: ID!): Product
  order(id: ID!): Order
  tenant(id: ID!): Tenant
}

type User @key(fields: "id") {
  id: ID!
  email: String!
  orders: [Order!]! @requires(fields: "id")
  tenant: Tenant! @requires(fields: "tenantId")
}

type Product @key(fields: "id") {
  id: ID!
  name: String!
  inventory: Inventory! @requires(fields: "id")
}
```

#### **gRPC for Internal Communication**
```protobuf
// user.proto
service UserService {
  rpc GetUser(GetUserRequest) returns (UserResponse);
  rpc CreateUser(CreateUserRequest) returns (UserResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UserResponse);
  rpc ValidateUser(ValidateUserRequest) returns (ValidationResponse);
}

message User {
  string id = 1;
  string email = 2;
  string username = 3;
  string tenant_id = 4;
  int64 created_at = 5;
  int64 updated_at = 6;
}
```

### 2. **Asynchronous Communication**

#### **Event-Driven Architecture**
```typescript
// Event Bus Implementation
interface EventBus {
  publish<T>(event: DomainEvent<T>): Promise<void>
  subscribe<T>(eventType: string, handler: EventHandler<T>): void
  subscribeToPattern(pattern: string, handler: EventHandler<any>): void
}

// Domain Events
interface DomainEvent<T> {
  id: string
  type: string
  version: number
  timestamp: Date
  aggregateId: string
  tenantId: string
  data: T
  metadata?: Record<string, any>
}

// Event Handlers
class OrderEventHandler {
  @EventHandler('UserCreated')
  async handleUserCreated(event: DomainEvent<UserCreatedData>): Promise<void> {
    // Initialize user's order history
    await this.orderService.initializeUserOrderHistory(event.data.userId);
  }
  
  @EventHandler('ProductInventoryUpdated')
  async handleInventoryUpdate(event: DomainEvent<InventoryUpdatedData>): Promise<void> {
    // Check for pending orders that can now be fulfilled
    await this.orderService.checkPendingOrders(event.data.productId);
  }
}
```

#### **SAGA Pattern for Distributed Transactions**
```typescript
// Order Processing Saga
class OrderProcessingSaga {
  @SagaStart
  @EventHandler('OrderCreated')
  async handleOrderCreated(event: DomainEvent<OrderCreatedData>): Promise<void> {
    const { orderId, userId, items } = event.data;
    
    // Step 1: Reserve inventory
    await this.commandBus.send(new ReserveInventoryCommand(orderId, items));
  }
  
  @EventHandler('InventoryReserved')
  async handleInventoryReserved(event: DomainEvent<InventoryReservedData>): Promise<void> {
    const { orderId } = event.data;
    
    // Step 2: Process payment
    await this.commandBus.send(new ProcessPaymentCommand(orderId));
  }
  
  @EventHandler('PaymentProcessed')
  async handlePaymentProcessed(event: DomainEvent<PaymentProcessedData>): Promise<void> {
    const { orderId } = event.data;
    
    // Step 3: Fulfill order
    await this.commandBus.send(new FulfillOrderCommand(orderId));
  }
  
  // Compensation handlers for rollback
  @EventHandler('PaymentFailed')
  async handlePaymentFailed(event: DomainEvent<PaymentFailedData>): Promise<void> {
    const { orderId } = event.data;
    
    // Compensate: Release inventory
    await this.commandBus.send(new ReleaseInventoryCommand(orderId));
  }
}
```

## Data Access Layer

### **Database Abstraction Pattern**
```typescript
// Repository Pattern with Database Abstraction
interface Repository<T> {
  findById(id: string): Promise<T | null>
  findMany(query: QueryOptions): Promise<T[]>
  create(entity: Partial<T>): Promise<T>
  update(id: string, updates: Partial<T>): Promise<T>
  delete(id: string): Promise<void>
  count(query: QueryOptions): Promise<number>
}

// Database-agnostic implementation
class UserRepository implements Repository<User> {
  constructor(private db: DatabaseAdapter) {}
  
  async findById(id: string): Promise<User | null> {
    return await this.db.findOne('users', { id });
  }
  
  async findByEmail(email: string): Promise<User | null> {
    return await this.db.findOne('users', { email });
  }
  
  async findByTenant(tenantId: string): Promise<User[]> {
    return await this.db.findMany('users', { tenantId });
  }
}

// Database adapters for different databases
interface DatabaseAdapter {
  findOne(collection: string, query: any): Promise<any>
  findMany(collection: string, query: any): Promise<any[]>
  insertOne(collection: string, document: any): Promise<any>
  updateOne(collection: string, query: any, update: any): Promise<any>
  deleteOne(collection: string, query: any): Promise<void>
}

class PostgreSQLAdapter implements DatabaseAdapter {
  constructor(private client: PostgreSQLClient) {}
  // Implementation using SQL queries
}

class MongoDBAdapter implements DatabaseAdapter {
  constructor(private client: MongoClient) {}
  // Implementation using MongoDB operations
}
```

### **CQRS Pattern**
```typescript
// Command Query Responsibility Segregation
interface CommandHandler<T> {
  handle(command: T): Promise<void>
}

interface QueryHandler<T, R> {
  handle(query: T): Promise<R>
}

// Commands (Write operations)
class CreateUserCommand {
  constructor(
    public readonly userData: CreateUserRequest,
    public readonly tenantId: string
  ) {}
}

class CreateUserCommandHandler implements CommandHandler<CreateUserCommand> {
  constructor(
    private userRepository: UserRepository,
    private eventBus: EventBus
  ) {}
  
  async handle(command: CreateUserCommand): Promise<void> {
    const user = await this.userRepository.create(command.userData);
    
    await this.eventBus.publish(new UserCreatedEvent({
      userId: user.id,
      email: user.email,
      tenantId: command.tenantId
    }));
  }
}

// Queries (Read operations)
class GetUserQuery {
  constructor(public readonly userId: string) {}
}

class GetUserQueryHandler implements QueryHandler<GetUserQuery, User> {
  constructor(private userReadModel: UserReadModel) {}
  
  async handle(query: GetUserQuery): Promise<User> {
    return await this.userReadModel.findById(query.userId);
  }
}
```

## Service Discovery and Load Balancing

### **Service Mesh Integration**
```yaml
# Istio Service Mesh Configuration
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts:
  - user-service
  http:
  - route:
    - destination:
        host: user-service
        subset: v1
      weight: 90
    - destination:
        host: user-service
        subset: v2
      weight: 10  # Canary deployment
  - fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s  # Chaos testing
```

### **Health Checks and Circuit Breakers**
```typescript
// Health check endpoint for each service
class HealthController {
  @Get('/health')
  async healthCheck(): Promise<HealthStatus> {
    const dbHealth = await this.checkDatabaseHealth();
    const externalServicesHealth = await this.checkExternalServices();
    
    return {
      status: dbHealth && externalServicesHealth ? 'healthy' : 'unhealthy',
      timestamp: new Date(),
      dependencies: {
        database: dbHealth ? 'up' : 'down',
        eventBus: externalServicesHealth ? 'up' : 'down'
      }
    };
  }
}

// Circuit breaker pattern
class CircuitBreaker {
  private failureCount = 0;
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
  private lastFailureTime = 0;
  
  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.timeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is open');
      }
    }
    
    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
}
```

## API Gateway and Security

### **Rate Limiting and Throttling**
```typescript
// Rate limiting middleware
class RateLimitMiddleware {
  constructor(private redis: RedisClient) {}
  
  async use(req: Request, res: Response, next: NextFunction): Promise<void> {
    const key = `rate_limit:${req.ip}:${req.path}`;
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      await this.redis.expire(key, 60); // 1 minute window
    }
    
    if (current > 100) { // 100 requests per minute
      res.status(429).json({ error: 'Rate limit exceeded' });
      return;
    }
    
    next();
  }
}
```

### **Authentication Middleware**
```typescript
// JWT authentication middleware
class AuthMiddleware {
  constructor(private authService: AuthService) {}
  
  async use(req: Request, res: Response, next: NextFunction): Promise<void> {
    const token = req.headers.authorization?.replace('Bearer ', '');
    
    if (!token) {
      res.status(401).json({ error: 'No token provided' });
      return;
    }
    
    try {
      const validation = await this.authService.validateToken(token);
      req.user = validation.user;
      req.tenant = validation.tenant;
      next();
    } catch (error) {
      res.status(401).json({ error: 'Invalid token' });
    }
  }
}
```

## Monitoring and Observability

### **Distributed Tracing**
```typescript
// OpenTelemetry integration
import { trace, context } from '@opentelemetry/api';

class UserService {
  private tracer = trace.getTracer('user-service');
  
  async createUser(userData: CreateUserRequest): Promise<User> {
    const span = this.tracer.startSpan('user.create');
    
    try {
      span.setAttributes({
        'user.email': userData.email,
        'user.tenant_id': userData.tenantId
      });
      
      const user = await this.userRepository.create(userData);
      
      span.setStatus({ code: trace.SpanStatusCode.OK });
      return user;
    } catch (error) {
      span.setStatus({ 
        code: trace.SpanStatusCode.ERROR, 
        message: error.message 
      });
      throw error;
    } finally {
      span.end();
    }
  }
}
```

This backend architecture provides a solid foundation for building scalable, maintainable SaaS applications with clear separation of concerns, strong data consistency patterns, and comprehensive observability.