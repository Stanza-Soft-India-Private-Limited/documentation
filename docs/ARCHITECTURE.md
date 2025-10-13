---
layout: default
title: Architecture
nav_order: 3
---

# Architecture

**Last Updated:** October 10, 2025

---

## Table of Contents

- [High-Level Architecture](#high-level-architecture)
- [Project Structure](#project-structure)
- [Module Organization](#module-organization)
- [Data Flow](#data-flow)
- [Key Design Patterns](#key-design-patterns)
- [Scalability Considerations](#scalability-considerations)

---

## High-Level Architecture

### System Components Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│         (Web App, Mobile App, Third-party Integrations)          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ HTTPS/REST
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  NestJS Application (Port 3000)                           │  │
│  │  - CORS, Helmet Security                                  │  │
│  │  - Rate Limiting (100 req/min)                            │  │
│  │  - JWT Authentication Guard                               │  │
│  │  - Request ID Middleware                                  │  │
│  │  - Performance Interceptors                               │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ↓               ↓               ↓
┌─────────────────┐ ┌──────────┐ ┌────────────────┐
│  AWS Cognito    │ │  Redis   │ │  Mux Video     │
│  (Auth)         │ │  (Cache) │ │  (Streaming)   │
└─────────────────┘ └──────────┘ └────────────────┘
         │               │
         │               │
         ↓               ↓
┌─────────────────────────────────────────────────────────────────┐
│                     Business Logic Layer                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Auth Module  │  │  User Module │  │  PYQ Module  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Mains Module │  │ Reels Module │  │ Graph Module │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ↓               ↓               ↓
┌─────────────────┐ ┌──────────┐ ┌────────────────┐
│  PostgreSQL     │ │  Neo4j   │ │  Redis         │
│  (Primary DB)   │ │  (Graph) │ │  (Queue)       │
│  - Users        │ │  - User  │ │  - BullMQ      │
│  - Questions    │ │  - Ques  │ │  - Sessions    │
│  - Sessions     │ │  - Rels  │ │  - Cache       │
└─────────────────┘ └──────────┘ └────────────────┘
```

### Polyglot Persistence Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                     Data Storage Strategy                       │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PostgreSQL (Relational)          Neo4j (Graph)                │
│  ├─ User Authentication           ├─ Learning Relationships    │
│  ├─ User Profiles                 ├─ Question Attempts         │
│  ├─ PYQ Questions                 ├─ User Progress Nodes       │
│  ├─ Mains Papers                  ├─ Topic Relationships       │
│  ├─ Video Metadata                ├─ Recommendation Graph      │
│  ├─ Sessions                      └─ Streak Tracking           │
│  └─ Audit Logs                                                 │
│                                                                 │
│  Redis (In-Memory)                                             │
│  ├─ API Response Cache                                         │
│  ├─ Session Storage                                            │
│  ├─ BullMQ Job Queue                                           │
│  └─ Rate Limiting Counters                                     │
└────────────────────────────────────────────────────────────────┘
```

---

## Project Structure

### Directory Tree

```
src/
├── app.module.ts                    # Root application module
├── app.controller.ts                # Root controller
├── app.service.ts                   # Root service
├── main.ts                          # Application bootstrap
│
├── modules/                         # Feature modules
│   ├── auth/                        # Authentication module
│   │   ├── auth.module.ts
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── dto/                     # Data Transfer Objects
│   │   ├── guards/                  # JWT auth guards
│   │   ├── strategies/              # Passport strategies
│   │   └── services/                # Session service
│   │
│   ├── user/                        # User management
│   │   ├── user.module.ts
│   │   ├── user.controller.ts
│   │   ├── user.service.ts
│   │   └── dto/
│   │
│   ├── pyq/                         # Previous Year Questions
│   │   ├── pyq.module.ts
│   │   ├── pyq.controller.ts
│   │   ├── pyq.service.ts
│   │   └── dto/
│   │
│   ├── mains/                       # Mains papers
│   │   ├── mains.module.ts
│   │   ├── mains.controller.ts
│   │   ├── mains.service.ts
│   │   └── dto/
│   │
│   ├── reels/                       # Video reels
│   │   ├── reels.module.ts
│   │   ├── reels.controller.ts
│   │   ├── reels.service.ts
│   │   └── dto/
│   │
│   ├── chat/                        # Chat module (scaffolded)
│   │   ├── chat.module.ts
│   │   ├── chat.controller.ts
│   │   ├── chat.service.ts
│   │   └── dto/
│   │
│   └── health/                      # Health monitoring
│       ├── health.module.ts
│       └── health.controller.ts
│
├── graph/                           # Neo4j graph database layer
│   ├── graph.module.ts
│   ├── types/                       # TypeScript types
│   │   ├── nodes.types.ts
│   │   └── relationships.types.ts
│   ├── dto/                         # Graph DTOs
│   ├── repositories/                # Data access layer
│   │   ├── user.repository.ts
│   │   ├── question.repository.ts
│   │   ├── assessment.repository.ts
│   │   └── session.repository.ts
│   ├── services/                    # Business logic
│   │   └── progress.service.ts
│   └── controllers/
│       └── progress.controller.ts
│
├── neo4j/                           # Neo4j driver service
│   ├── neo4j.module.ts
│   └── neo4j.service.ts
│
├── database/                        # PostgreSQL/Prisma
│   ├── prisma.module.ts
│   └── prisma.service.ts
│
├── common/                          # Shared utilities
│   ├── decorators/                  # Custom decorators
│   │   ├── current-user.decorator.ts
│   │   ├── public.decorator.ts
│   │   ├── roles.decorator.ts
│   │   ├── cache-ttl.decorator.ts
│   │   └── api-paginated.decorator.ts
│   │
│   ├── guards/                      # Route guards
│   │   └── roles.guard.ts
│   │
│   ├── filters/                     # Exception filters
│   │   └── exception.filter.ts
│   │
│   ├── interceptors/                # Request/response interceptors
│   │   ├── performance.interceptor.ts
│   │   └── cache.interceptor.ts
│   │
│   ├── middleware/                  # Express middleware
│   │   └── request-id.middleware.ts
│   │
│   ├── dto/                         # Shared DTOs
│   │   └── pagination.dto.ts
│   │
│   ├── interfaces/                  # Shared interfaces
│   │   └── pagination.interface.ts
│   │
│   ├── utils/                       # Utility functions
│   │   └── pagination.util.ts
│   │
│   ├── services/                    # Shared services
│   │   └── cache.service.ts
│   │
│   ├── http-client/                 # HTTP client module
│   │   └── http-client.module.ts
│   │
│   └── shared/                      # Global shared modules
│       ├── cache.module.ts
│       └── queue.module.ts
│
├── config/                          # Configuration files
│   ├── app.config.ts
│   ├── database.config.ts
│   ├── neo4j.config.ts
│   ├── redis.config.ts
│   └── mux.config.ts
│
└── generated/                       # Auto-generated code
    ├── client/                      # Prisma client
    ├── enums/                       # Enum types
    └── nestjs-dto/                  # Auto-generated DTOs
```

### Key Directories Explained

| Directory    | Purpose                                        | When to Use                     |
| ------------ | ---------------------------------------------- | ------------------------------- |
| `modules/`   | Feature-specific code organized by domain      | Adding new features             |
| `graph/`     | Neo4j graph database layer (progress tracking) | Graph queries and relationships |
| `neo4j/`     | Low-level Neo4j driver wrapper                 | Direct database operations      |
| `common/`    | Shared utilities, decorators, guards           | Cross-cutting concerns          |
| `config/`    | Environment-based configuration                | Adding new config values        |
| `generated/` | Prisma-generated code                          | Never edit manually             |

---

## Module Organization

### NestJS Module Architecture

Each feature is organized as a self-contained NestJS module following this pattern:

```
feature-module/
├── feature.module.ts          # Module definition
├── feature.controller.ts      # REST endpoints
├── feature.service.ts         # Business logic
├── dto/                       # Request/response DTOs
│   ├── create-feature.dto.ts
│   ├── update-feature.dto.ts
│   └── feature-filters.dto.ts
└── interfaces/                # TypeScript interfaces
    └── feature.interface.ts
```

### Module Dependencies

```
AppModule (Root)
├── ConfigModule (Global)
├── ThrottlerModule (Global)
├── PrismaModule
├── Neo4jModule
├── HttpClientModule
├── CacheModule
├── QueueModule
├── GraphModule
│   ├── Neo4jModule (imported)
│   ├── UserRepository
│   ├── QuestionRepository
│   ├── SessionRepository
│   ├── AssessmentRepository
│   └── ProgressService
│
├── AuthModule
│   ├── PrismaModule (imported)
│   ├── JwtModule
│   ├── AuthService
│   └── SessionsService
│
├── UserModule
│   └── PrismaModule (imported)
│
├── PyqModule
│   └── PrismaModule (imported)
│
├── MainsModule
│   └── PrismaModule (imported)
│
├── ReelsModule
│   └── PrismaModule (imported)
│
├── ChatModule (scaffolded)
│
└── HealthModule
    ├── PrismaModule (imported)
    └── Neo4jModule (imported)
```

---

## Data Flow

### 1. Authentication Flow

```
User (Client)
  │
  │ 1. Click "Sign in with Google"
  ↓
Google OAuth
  │
  │ 2. OAuth tokens
  ↓
AWS Cognito
  │
  │ 3. JWT token
  ↓
POST /api/v1/auth/cognito/exchange
  │
  │ 4. Verify token with AWS
  ↓
AuthService
  │
  │ 5. Check/create user in DB
  ↓
PostgreSQL (UserAuth table)
  │
  │ 6. Create session
  ↓
SessionsService
  │
  │ 7. Generate backend JWT
  ↓
Response: { accessToken, refreshToken, user }
```

### 2. Authenticated Request Flow

```
Client Request with JWT
  │
  │ Authorization: Bearer <token>
  ↓
JwtAuthGuard (Global)
  │
  │ Validate token signature
  ↓
@CurrentUser() decorator
  │
  │ Extract user from token
  ↓
Controller Method
  │
  │ Business logic
  ↓
Service Layer
  │
  ├─→ PostgreSQL (via Prisma)
  │   └─→ Return transactional data
  │
  ├─→ Neo4j (via Neo4jService)
  │   └─→ Return graph relationships
  │
  └─→ Redis (via CacheService)
      └─→ Return cached data
  │
  ↓
Response Transformation
  │
  ↓
Client receives JSON response
```

### 3. Question Attempt Flow (Polyglot Persistence)

```
User attempts question
  │
  ↓
POST /api/v1/progress/questions/:id/attempt
  │
  │ { userId, questionId, selectedOption, isCorrect }
  ↓
ProgressController
  │
  ↓
QuestionRepository (Neo4j)
  │
  │ 1. Create ATTEMPTED relationship
  │    Properties: attemptNumber, timestamp, isCorrect
  ↓
Neo4j Database
  │ (User)-[ATTEMPTED {attemptNumber: 3}]->(Question)
  │
  │ 2. Trigger event: QuestionAttempted
  ↓
Event Emitter (Redis Pub/Sub)
  │
  ↓
[Future] Update user statistics in PostgreSQL
  │
  ↓
Response: { success: true }
```

### 4. Caching Flow

```
GET /api/v1/pyq?subject=History
  │
  ↓
CacheInterceptor
  │
  │ Check Redis for cached response
  ↓
Redis Cache Hit?
  │
  ├─ YES → Return cached data (fast)
  │
  └─ NO  → Continue to controller
             │
             ↓
         PyqController
             │
             ↓
         PyqService
             │
             ↓
         PostgreSQL (via Prisma)
             │
             ↓
         CacheInterceptor
             │
             │ Store response in Redis (TTL: 60s)
             ↓
         Return data to client
```

---

## Key Design Patterns

### 1. Repository Pattern (Graph Module)

**Purpose:** Separate data access logic from business logic

```typescript
// Instead of direct queries in services:
❌ BAD:
async getUserProgress(userId: string) {
  const result = await this.neo4j.query(`
    MATCH (u:User {id: $userId})...
  `);
  // Complex query logic in service
}

✅ GOOD:
async getUserProgress(userId: string) {
  const user = await this.userRepo.getUserById(userId);
  const stats = await this.questionRepo.getUserQuestionStats(userId);
  const sessions = await this.sessionRepo.getWeeklySummary(userId);
  return { user, stats, sessions };
}
```

**Benefits:**

- Testable (mock repositories)
- Reusable queries
- Clear separation of concerns

### 2. DTO Pattern (Data Transfer Objects)

**Purpose:** Validate and transform incoming data

```typescript
// Auto-generated from Prisma schema
export class CreatePYQPaperDto {
  @IsString()
  @IsNotEmpty()
  examName: string;

  @IsInt()
  @Min(2000)
  @Max(2024)
  year: number;

  @IsEnum(PYQDifficulty)
  difficulty: PYQDifficulty;

  // ... more fields with validation
}
```

**Benefits:**

- Type safety
- Automatic validation
- Swagger documentation
- Clear API contracts

### 3. Dependency Injection

**Purpose:** Loose coupling and testability

```typescript
@Injectable()
export class PyqService {
  constructor(
    private readonly prisma: PrismaService, // Injected
    private readonly cacheService: CacheService, // Injected
  ) {}
}
```

**Benefits:**

- Easy to mock in tests
- No tight coupling to implementations
- Framework manages lifecycle

### 4. Guard Pattern (Authorization)

**Purpose:** Protect routes based on authentication and roles

```typescript
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(Role.ADMIN, Role.SUPER_ADMIN)
@Delete('profile/:id')
async deleteProfile(@Param('id') id: string) {
  // Only admins can reach here
}
```

**Benefits:**

- Declarative security
- Reusable across endpoints
- Clear authorization rules

### 5. Interceptor Pattern

**Purpose:** Transform requests/responses globally

```typescript
@UseInterceptors(CacheInterceptor, PerformanceInterceptor)
@Get()
async findAll() {
  // Automatic caching and performance logging
}
```

**Use Cases:**

- Response caching
- Performance monitoring
- Logging
- Response transformation

### 6. Pagination Pattern

**Purpose:** Standardized pagination across all list endpoints

```typescript
// Common pagination DTO
export class PaginationDto {
  @IsOptional()
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 10;
}

// Response format
interface PaginatedResult<T> {
  data: T[];
  meta: {
    total: number;
    page: number;
    limit: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}
```

---

## Scalability Considerations

### Horizontal Scaling

**Current Architecture Supports:**

- ✅ Stateless API servers (JWT-based auth, no server sessions)
- ✅ Database connection pooling
- ✅ Redis for shared session storage
- ✅ Docker containerization ready

**To Scale:**

1. Deploy multiple API server instances behind load balancer
2. Each instance connects to same PostgreSQL, Neo4j, and Redis
3. AWS ALB already configured: `http://upsc-alb-590268270.ap-south-1.elb.amazonaws.com`

### Database Scaling

**PostgreSQL:**

- Read replicas for read-heavy operations
- Connection pooling (configured in Prisma)
- Indexes on frequently queried columns

**Neo4j:**

- Causal clustering for high availability
- Read replicas for recommendation queries
- Current: Single instance (sufficient for MVP)

**Redis:**

- Redis Cluster for high availability
- Separate instances for cache vs. queue
- Current: Single instance

### Performance Optimizations

**Implemented:**

- Response caching (60s TTL)
- GZIP compression
- Pagination on all list endpoints
- Database query optimization with indexes

**Future Improvements:**

- CDN for static assets
- Database query result caching
- GraphQL for flexible data fetching
- WebSocket for real-time features

---

## Security Architecture

### Layers of Security

```
┌──────────────────────────────────────────────────┐
│ 1. Network Layer                                 │
│    - HTTPS/TLS encryption                        │
│    - CORS restrictions                           │
└──────────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────┐
│ 2. Application Layer                             │
│    - Helmet (security headers)                   │
│    - Rate limiting (100 req/min)                 │
│    - Input validation (class-validator)          │
└──────────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────┐
│ 3. Authentication Layer                          │
│    - JWT verification                            │
│    - AWS Cognito integration                     │
│    - Session management                          │
└──────────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────┐
│ 4. Authorization Layer                           │
│    - Role-based access control                   │
│    - Resource-level permissions                  │
└──────────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────┐
│ 5. Data Layer                                    │
│    - Parameterized queries (Prisma)              │
│    - Encrypted connections                       │
│    - No sensitive data in logs                   │
└──────────────────────────────────────────────────┘
```

---

## Monitoring Strategy

### Health Checks

```
/api/v1/health           → Overall system health
/api/v1/health/db        → PostgreSQL connectivity
/api/v1/health/neo4j     → Neo4j connectivity
/api/v1/health/cognito   → AWS Cognito availability
```

### Logging Levels

- **Development:** error, warn, log
- **Production:** error, warn only

### Performance Metrics

- Request duration (via PerformanceInterceptor)
- Database query times
- Cache hit rates

---

## Future Architecture Enhancements

1. **Microservices:** Split into separate services (auth, content, progress)
2. **Event-Driven:** Full event sourcing with Kafka/RabbitMQ
3. **GraphQL:** Add GraphQL layer for flexible queries
4. **CQRS:** Separate read/write models for complex domains
5. **Service Mesh:** Istio for inter-service communication
6. **Observability:** Prometheus + Grafana for metrics

---

## Next Steps

- Read [DATABASE.md](./DATABASE.md) for detailed schema documentation
- See [FEATURES.md](./FEATURES.md) for module-specific architecture
- Check [DEPLOYMENT.md](./DEPLOYMENT.md) for production architecture
