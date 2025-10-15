# UPSC Exam Preparation Platform - Developer Documentation

**Last Updated:** October 10, 2025
**Version:** 1.0.0
**Status:** Active Development

---

## What Is This Project?

A NestJS-based backend API for a comprehensive UPSC Civil Services Examination preparation platform. The system uses polyglot persistence (PostgreSQL + Neo4j) to handle both transactional data and complex learning relationships, enabling intelligent question recommendations and progress tracking.

---

## Current Status

- **Development Stage:** Active Development (MVP Complete)
- **Version:** 1.0.0
- **Deployment Status:** Deployed on AWS ELB
- **Production URL:** http://upsc-alb-590268270.ap-south-1.elb.amazonaws.com

### What's Complete

✅ Core authentication with AWS Cognito (Google & Apple)
✅ User profile management
✅ PYQ (Previous Year Questions) CRUD with filtering
✅ Mains papers management
✅ Video reels upload and streaming via Mux
✅ Health monitoring endpoints
✅ Graph database schema for progress tracking

### What's In Progress

🚧 Progress tracking business logic
🚧 Recommendation engine algorithms

### What's Planned

📋 Psychometric assessment implementation
📋 AI chat tutor integration
📋 Advanced analytics dashboards

---

## Key Technologies

| Layer             | Technology      | Version |
| ----------------- | --------------- | ------- |
| **Framework**     | NestJS          | 11.0.1  |
| **Runtime**       | Node.js         | 22.17.0 |
| **Language**      | TypeScript      | 5.7.3   |
| **Relational DB** | PostgreSQL      | 16+     |
| **ORM**           | Prisma          | 6.15.0  |
| **Graph DB**      | Neo4j           | 5.x     |
| **Cache/Queue**   | Redis           | 7.x     |
| **Auth**          | AWS Cognito     | -       |
| **Video**         | Mux             | 12.8.0  |
| **API Docs**      | Swagger/OpenAPI | 3.0     |
| **Testing**       | Jest            | 29.7.0  |

---

## Quick Links

### For Product Managers

- [Product Documentation](./PROJECT_DOCUMENTATION.md) - Feature overview and roadmap

### For Developers

- [Architecture Guide](./ARCHITECTURE.md) - System design and patterns
- [API Reference](./API.md) - Endpoint documentation

### Additional Resources

- [CLAUDE.md](../CLAUDE.md) - Comprehensive technical specification

---

## Quick Start

```bash
# 1. Install dependencies
npm install

# 2. Start Docker services
docker-compose up -d

# 3. Setup databases
npm run db:generate
npm run db:migrate
npm run graph:init

# 4. Start development server
npm run start:dev

# 5. Access Swagger docs
open http://localhost:3000/api/docs
```

**Default Swagger Credentials:**
Username: `admin`
Password: `password` (change in production via env vars)

---

## Project Structure Overview

```
backend/
├── src/
│   ├── modules/          # Feature modules (auth, user, pyq, etc.)
│   ├── graph/            # Neo4j graph database layer
│   ├── neo4j/            # Neo4j driver service
│   ├── common/           # Shared utilities and decorators
│   ├── config/           # Configuration files
│   ├── database/         # Prisma setup
│   └── generated/        # Auto-generated DTOs and types
├── prisma/               # Database schema and migrations
├── docs/                 # Documentation (you are here)
├── test/                 # E2E tests
└── seed/                 # Database seed data
```

---

## Environment Variables

Create a `.env` file in the root directory:

```env
# App Configuration
NODE_ENV=development
PORT=3000
API_VERSION=v1

# AWS Cognito
AWS_COGNITO_USER_POOL_ID=your-pool-id
AWS_COGNITO_CLIENT_ID=your-client-id
AWS_REGION=us-east-1

# PostgreSQL
DATABASE_URL="postgresql://user:password@localhost:5432/upsc_db"

# Neo4j
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=12345678
NEO4J_DATABASE=neo4j

# Redis
REDIS_URL=redis://localhost:6379

# Mux (Video Platform)
MUX_TOKEN_ID=your-mux-token-id
MUX_TOKEN_SECRET=your-mux-token-secret

# Swagger Authentication
SWAGGER_USERNAME=admin
SWAGGER_PASSWORD=your-secure-password
SWAGGER_SESSION_SECRET=your-session-secret

# CORS
CORS_ORIGIN=http://localhost:3000,http://localhost:5173
```

See [SETUP.md](./SETUP.md) for detailed configuration instructions.

---

## Available Commands

### Development

```bash
npm run start:dev      # Start with hot-reload
npm run start:debug    # Start with debugger
npm run build          # Build for production
npm run start:prod     # Start production build
```

### Database Management

```bash
npm run db:generate    # Generate Prisma client
npm run db:migrate     # Create and apply migration
npm run db:deploy      # Apply migrations (production)
npm run db:reset       # Reset database
npm run db:studio      # Open Prisma Studio GUI
npm run db:seed        # Seed sample data
```

### Graph Database

```bash
npm run graph:init     # Initialize Neo4j schema
npm run graph:seed     # Seed graph data
npm run graph:reset    # Clear all graph data
npm run graph:studio   # Open Neo4j Browser
```

### Code Quality

```bash
npm run lint           # Run ESLint
npm run format         # Format with Prettier
npm run test           # Run unit tests
npm run test:cov       # Run with coverage
npm run test:e2e       # Run E2E tests
```

### Docker

```bash
docker-compose up -d           # Start all services
docker-compose down            # Stop all services
docker-compose logs -f         # Follow all logs
docker-compose logs -f neo4j   # Follow Neo4j logs
docker-compose restart neo4j   # Restart Neo4j
```

---

## Key Architectural Decisions

### 1. Polyglot Persistence

- **PostgreSQL:** Core transactional data (users, questions, sessions)
- **Neo4j:** Learning relationships and recommendation graphs
- **Redis:** Caching and session storage

**Why:** Different data access patterns require different data models. Relational for CRUD, graph for complex relationships.

### 2. AWS Cognito for Authentication

- **Federated identity** via Google and Apple
- **No password storage** in application database
- **JWT tokens** managed by AWS

**Why:** Secure, scalable authentication without managing credentials. Social login improves user experience.

### 3. Modular Architecture

- Each domain (auth, user, pyq, etc.) is a self-contained module
- Clear separation of concerns
- Easy to test and maintain

**Why:** Scalability, maintainability, and team collaboration.

---

## Security Considerations

- **Authentication:** All endpoints (except health and auth) require valid JWT
- **CORS:** Restricted origins configured via environment variables
- **Rate Limiting:** 100 requests per minute per user
- **Helmet:** Security headers enabled
- **Input Validation:** All DTOs validated with class-validator
- **SQL Injection:** Prevented via Prisma parameterized queries

---

## Testing Strategy

```bash
# Unit tests for services and repositories
npm run test

# E2E tests for API endpoints
npm run test:e2e

# Coverage report
npm run test:cov
```

Target: 80% code coverage minimum

---

## Debugging

### VS Code Launch Configuration

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach by Process ID",
      "processId": "${command:PickProcess}",
      "request": "attach",
      "skipFiles": ["<node_internals>/**"],
      "type": "node"
    },
    {
      "type": "node-terminal",
      "name": "Run Script: start:debug",
      "request": "launch",
      "command": "npm run start:debug",
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

Then press F5 to start debugging.

---

## Common Issues & Solutions

### Issue: Prisma Client Out of Sync

```bash
npm run db:generate
```

### Issue: Neo4j Connection Timeout

Check if Neo4j is running:

```bash
docker-compose ps neo4j
docker-compose logs neo4j
```

### Issue: Port Already in Use

```bash
# Find process on port 3000
lsof -ti:3000 | xargs kill
```

### Issue: Database Migration Conflicts

```bash
npm run db:reset  # Warning: Deletes all data!
```

---

## Performance Considerations

- **Caching:** API responses cached for 60 seconds (configurable)
- **Connection Pooling:** PostgreSQL connection pool size: 10
- **Pagination:** All list endpoints support page/limit parameters
- **Compression:** GZIP compression enabled for responses
- **Query Optimization:** Indexes on frequently queried columns

---

## Monitoring & Logging

### Health Check Endpoints

```
GET /api/v1/health           # Overall health
GET /api/v1/health/db        # PostgreSQL health
GET /api/v1/health/neo4j     # Neo4j health
GET /api/v1/health/cognito   # AWS Cognito health
```

### Logging

- **Development:** All log levels (error, warn, log)
- **Production:** Error and warn only
- **Format:** JSON structured logs

---

## Contributing

Before contributing, please read:

1. [CONTRIBUTING.md](./CONTRIBUTING.md) - Development guidelines
2. [CLAUDE.md](../CLAUDE.md) - Technical specifications
3. Code style: ESLint + Prettier (runs on pre-commit)

### Workflow

1. Create feature branch from `main`
2. Make changes with tests
3. Run `npm run lint` and `npm run test`
4. Submit pull request
5. Wait for code review

---

## Support & Resources

- **Bug Reports:** GitHub Issues
- **Technical Questions:** Check [CLAUDE.md](../CLAUDE.md)
- **Feature Requests:** Discuss with product team first

---

## License

MIT License - See LICENSE file for details

---

**Next Steps:**

- Read [SETUP.md](./SETUP.md) for detailed setup instructions
- Explore [ARCHITECTURE.md](./ARCHITECTURE.md) for system design
- Check [API.md](./API.md) for endpoint documentation
