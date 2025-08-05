# Infinity Server - SaaS Boilerplate Monorepo Specification

## Overview

A comprehensive TypeScript-based monorepo containing both API server and static file serving capabilities. The server provides essential SaaS functionality including authentication, database integration, logging, and testing infrastructure, while also serving static files for a basic web application. This unified approach allows for seamless full-stack development within a single codebase.

## Core Requirements

### 1. Authentication System
- **Method**: Username + Password only
- **Token System**: JWT (JSON Web Tokens)
- **Security**: Bcrypt password hashing
- **Session Management**: Stateless JWT-based authentication
- **Password Requirements**: Minimum 8 characters, configurable complexity

### 2. Database Integration
- **Primary Database**: PostgreSQL
- **ORM**: Prisma (Type-safe database client)
- **Connection Pooling**: Built-in Prisma connection management
- **Migrations**: Automated schema migrations
- **Multi-tenancy**: Database-level tenant isolation

### 3. Logging System
- **Library**: Winston
- **Log Levels**: error, warn, info, debug
- **Output Formats**: JSON for production, colorized console for development
- **Log Rotation**: Daily rotation with size limits
- **Request Logging**: HTTP request/response logging middleware

### 4. Static File Serving
- **Static Assets**: Serve HTML, CSS, JavaScript, images
- **SPA Support**: Single Page Application routing with fallback
- **Compression**: Gzip compression for static files
- **Caching**: Proper cache headers for static assets
- **Security**: Static file access controls and MIME type validation

### 5. Testing Infrastructure
- **Unit Testing**: Jest
- **Integration Testing**: Supertest with test database
- **E2E Testing**: Jest + Supertest full workflow tests
- **Test Coverage**: NYC/Istanbul for coverage reports
- **Test Database**: Separate PostgreSQL instance for testing

## Technical Stack

```
├── Framework: Express.js + TypeScript
├── Database: PostgreSQL + Prisma ORM
├── Authentication: JWT + bcrypt
├── Logging: Winston
├── Testing: Jest + Supertest
├── Validation: Zod
├── Environment: dotenv
├── Security: Helmet, CORS, rate limiting
├── Static Files: Express static middleware + compression
└── Process Management: Node.js with PM2 support
```

## Monorepo Project Structure

```
infinity-server/
├── packages/
│   ├── server/                    # Main API server + static file serving
│   │   ├── src/
│   │   │   ├── controllers/       # Route handlers
│   │   │   │   ├── auth.controller.ts
│   │   │   │   ├── user.controller.ts
│   │   │   │   ├── static.controller.ts
│   │   │   │   └── health.controller.ts
│   │   │   ├── middleware/        # Express middleware
│   │   │   │   ├── auth.middleware.ts
│   │   │   │   ├── logging.middleware.ts
│   │   │   │   ├── validation.middleware.ts
│   │   │   │   ├── static.middleware.ts
│   │   │   │   └── error.middleware.ts
│   │   │   ├── services/          # Business logic
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── user.service.ts
│   │   │   │   ├── token.service.ts
│   │   │   │   └── static.service.ts
│   │   │   ├── models/            # Prisma models and types
│   │   │   │   ├── user.model.ts
│   │   │   │   └── index.ts
│   │   │   ├── routes/            # Route definitions
│   │   │   │   ├── api/           # API routes
│   │   │   │   │   ├── auth.routes.ts
│   │   │   │   │   ├── user.routes.ts
│   │   │   │   │   └── index.ts
│   │   │   │   ├── static.routes.ts
│   │   │   │   └── index.ts
│   │   │   ├── utils/             # Utility functions
│   │   │   │   ├── logger.ts
│   │   │   │   ├── validator.ts
│   │   │   │   └── constants.ts
│   │   │   ├── config/            # Configuration
│   │   │   │   ├── database.ts
│   │   │   │   ├── jwt.ts
│   │   │   │   ├── static.ts
│   │   │   │   └── server.ts
│   │   │   ├── types/             # TypeScript definitions
│   │   │   │   ├── auth.types.ts
│   │   │   │   ├── user.types.ts
│   │   │   │   └── index.ts
│   │   │   └── app.ts             # Express app setup
│   │   ├── public/                # Static files directory
│   │   │   ├── index.html
│   │   │   ├── css/
│   │   │   ├── js/
│   │   │   ├── images/
│   │   │   └── assets/
│   │   ├── tests/                 # Test files
│   │   │   ├── unit/
│   │   │   ├── integration/
│   │   │   └── e2e/
│   │   ├── prisma/                # Database schema and migrations
│   │   │   ├── schema.prisma
│   │   │   └── migrations/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── jest.config.js
│   │   ├── .env.example
│   │   └── Dockerfile
│   └── shared/                    # Shared utilities across packages
│       ├── types/
│       ├── utils/
│       └── constants/
├── plan/                          # Project specifications
├── package.json                   # Root package.json with workspaces
├── tsconfig.json                  # Root TypeScript config
└── README.md
```

## Database Schema

### Core Tables

```sql
-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Sessions/Tokens (optional for token blacklisting)
CREATE TABLE user_sessions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    token_jti VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Audit logs
CREATE TABLE audit_logs (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    action VARCHAR(100) NOT NULL,
    resource VARCHAR(100),
    resource_id VARCHAR(100),
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Prisma Schema

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id           Int      @id @default(autoincrement())
  username     String   @unique
  passwordHash String   @map("password_hash")
  email        String?  @unique
  isActive     Boolean  @default(true) @map("is_active")
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")
  
  sessions     UserSession[]
  auditLogs    AuditLog[]
  
  @@map("users")
}

model UserSession {
  id        Int      @id @default(autoincrement())
  userId    Int      @map("user_id")
  tokenJti  String   @unique @map("token_jti")
  expiresAt DateTime @map("expires_at")
  createdAt DateTime @default(now()) @map("created_at")
  
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("user_sessions")
}

model AuditLog {
  id         Int      @id @default(autoincrement())
  userId     Int?     @map("user_id")
  action     String
  resource   String?
  resourceId String?  @map("resource_id")
  ipAddress  String?  @map("ip_address")
  userAgent  String?  @map("user_agent")
  createdAt  DateTime @default(now()) @map("created_at")
  
  user       User?    @relation(fields: [userId], references: [id])
  
  @@map("audit_logs")
}
```

## API Endpoints

### Authentication Routes

```typescript
POST   /api/auth/register     # User registration
POST   /api/auth/login        # User login
POST   /api/auth/logout       # User logout (token blacklist)
POST   /api/auth/refresh      # Token refresh
GET    /api/auth/me          # Get current user profile
```

### User Management Routes

```typescript
GET    /api/users            # Get all users (admin)
GET    /api/users/:id        # Get specific user
PUT    /api/users/:id        # Update user
DELETE /api/users/:id        # Delete user
PATCH  /api/users/:id/status # Activate/deactivate user
```

### System Routes

```typescript
GET    /api/health           # Health check
GET    /api/version          # API version info
```

## Request/Response Schemas

### Authentication

```typescript
// Register Request
interface RegisterRequest {
  username: string;
  password: string;
  email?: string;
}

// Login Request
interface LoginRequest {
  username: string;
  password: string;
}

// Auth Response
interface AuthResponse {
  success: boolean;
  data: {
    user: {
      id: number;
      username: string;
      email?: string;
      isActive: boolean;
      createdAt: string;
    };
    token: string;
    expiresAt: string;
  };
}

// Error Response
interface ErrorResponse {
  success: false;
  error: {
    code: string;
    message: string;
    details?: any;
  };
}
```

## Security Implementation

### Password Security
```typescript
// Password hashing with bcrypt
const SALT_ROUNDS = 12;
const hashPassword = (password: string) => bcrypt.hash(password, SALT_ROUNDS);
const comparePassword = (password: string, hash: string) => bcrypt.compare(password, hash);
```

### JWT Configuration
```typescript
interface JWTConfig {
  secret: string;
  expiresIn: string; // '24h'
  issuer: string;
  audience: string;
}

interface JWTPayload {
  sub: string; // user ID
  username: string;
  iat: number;
  exp: number;
  jti: string; // JWT ID for blacklisting
}
```

### Security Middleware
- **Helmet**: Security headers
- **CORS**: Cross-origin requests
- **Rate Limiting**: Request throttling
- **Input Validation**: Zod schema validation
- **SQL Injection Protection**: Prisma ORM parameterized queries

## Logging Configuration

### Log Structure
```typescript
interface LogEntry {
  timestamp: string;
  level: 'error' | 'warn' | 'info' | 'debug';
  message: string;
  meta?: {
    userId?: number;
    requestId?: string;
    ipAddress?: string;
    userAgent?: string;
    duration?: number;
    statusCode?: number;
    method?: string;
    url?: string;
    error?: Error;
  };
}
```

### Log Levels by Environment
- **Development**: debug, info, warn, error
- **Production**: info, warn, error
- **Testing**: error only

## Testing Strategy

### Unit Tests
- **Controllers**: Mock services, test request/response handling
- **Services**: Mock database, test business logic
- **Middleware**: Test authentication, validation, error handling
- **Utilities**: Test helper functions

### Integration Tests
- **Database**: Test Prisma operations with test database
- **API Routes**: Test complete request/response cycles
- **Authentication Flow**: Test login, token validation, logout

### E2E Tests
- **Complete User Journey**: Registration → Login → Protected Routes → Logout
- **Error Scenarios**: Invalid credentials, expired tokens, validation errors

### Test Configuration
```typescript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/types/**/*.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
};
```

## Environment Configuration

### Required Environment Variables
```bash
# Server
NODE_ENV=development
PORT=3000
HOST=localhost

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/infinity_server
TEST_DATABASE_URL=postgresql://user:password@localhost:5432/infinity_server_test

# JWT
JWT_SECRET=your-super-secret-jwt-key
JWT_EXPIRES_IN=24h
JWT_ISSUER=infinity-server
JWT_AUDIENCE=infinity-client

# Logging
LOG_LEVEL=info
LOG_FILE_PATH=./logs/app.log
```

## Static File Serving Configuration

### Express Static Middleware Setup
```typescript
// Static file serving configuration
app.use('/static', express.static(path.join(__dirname, '../public'), {
  maxAge: '1d',
  etag: true,
  index: false,
  dotfiles: 'deny'
}));

// SPA fallback for client-side routing
app.get('*', (req, res, next) => {
  // Skip API routes
  if (req.path.startsWith('/api/')) {
    return next();
  }
  
  // Serve index.html for SPA routes
  res.sendFile(path.join(__dirname, '../public/index.html'));
});
```

### Static File Structure
```
public/
├── index.html           # Main HTML file
├── css/
│   ├── main.css        # Main stylesheet
│   └── components.css  # Component styles
├── js/
│   ├── main.js         # Main JavaScript
│   ├── auth.js         # Authentication logic
│   └── api.js          # API client
├── images/
│   ├── logo.png
│   └── favicon.ico
└── assets/
    ├── fonts/
    └── icons/
```

## Development Commands

```bash
# Development
npm run dev              # Start development server with hot reload
npm run build           # Build TypeScript to JavaScript
npm run start           # Start production server
npm run dev:server      # Start only the server package
npm run build:server    # Build only the server package

# Database
npm run db:migrate      # Run database migrations
npm run db:generate     # Generate Prisma client
npm run db:seed         # Seed database with initial data
npm run db:reset        # Reset database (dev only)

# Static Files
npm run build:static    # Build/optimize static files
npm run watch:static    # Watch static files for changes

# Testing
npm run test            # Run all tests
npm run test:unit       # Run unit tests only
npm run test:integration # Run integration tests
npm run test:e2e        # Run end-to-end tests
npm run test:coverage   # Run tests with coverage report
npm run test:watch      # Run tests in watch mode

# Code Quality
npm run lint            # Run ESLint
npm run format          # Run Prettier
npm run type-check      # Run TypeScript compiler check
```

## Performance Considerations

### Database Optimization
- Connection pooling (max 20 connections)
- Query optimization with proper indexes
- Prepared statements via Prisma
- Database query logging in development

### Memory Management
- JWT token caching
- Connection reuse
- Proper error handling to prevent memory leaks

### Monitoring Endpoints
```typescript
GET /api/health/live     # Liveness probe
GET /api/health/ready    # Readiness probe  
GET /api/metrics         # Application metrics
```

## Deployment Configuration

### Docker Support
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY dist/ ./dist/
EXPOSE 3000
CMD ["node", "dist/app.js"]
```

### Health Checks
- Database connectivity check
- Memory usage monitoring
- Response time monitoring
- Error rate tracking

This specification provides a complete foundation for building a production-ready SaaS API server with all essential features including robust authentication, comprehensive testing, detailed logging, and seamless database integration.