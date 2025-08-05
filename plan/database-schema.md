# Database Schema & Integration Patterns

## Overview

PostgreSQL database schema designed for SaaS applications with user authentication, session management, and audit logging. Built with Prisma ORM for type-safe database operations.

## Database Configuration

### Connection Settings
```env
DATABASE_URL=postgresql://username:password@localhost:5432/infinity_server
TEST_DATABASE_URL=postgresql://username:password@localhost:5432/infinity_server_test

# Connection Pool Settings
DB_POOL_MIN=2
DB_POOL_MAX=20
DB_POOL_TIMEOUT=30000
DB_POOL_IDLE_TIMEOUT=600000
```

### Prisma Configuration
```prisma
generator client {
  provider = "prisma-client-js"
  output   = "./generated/client"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

## Core Schema Definition

### 1. Users Table

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for performance
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email) WHERE email IS NOT NULL;
CREATE INDEX idx_users_active ON users(is_active) WHERE is_active = true;
CREATE INDEX idx_users_created_at ON users(created_at);
```

**Prisma Model:**
```prisma
model User {
  id           Int      @id @default(autoincrement())
  username     String   @unique @db.VarChar(255)
  passwordHash String   @map("password_hash") @db.VarChar(255)
  email        String?  @unique @db.VarChar(255)
  isActive     Boolean  @default(true) @map("is_active")
  createdAt    DateTime @default(now()) @map("created_at") @db.Timestamptz
  updatedAt    DateTime @updatedAt @map("updated_at") @db.Timestamptz
  
  // Relations
  sessions     UserSession[]
  auditLogs    AuditLog[]
  
  @@map("users")
  @@index([username])
  @@index([email])
  @@index([isActive])
  @@index([createdAt])
}
```

### 2. User Sessions Table

```sql
CREATE TABLE user_sessions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_jti VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for token validation and cleanup
CREATE INDEX idx_user_sessions_jti ON user_sessions(token_jti);
CREATE INDEX idx_user_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_user_sessions_expires_at ON user_sessions(expires_at);
CREATE INDEX idx_user_sessions_created_at ON user_sessions(created_at);
```

**Prisma Model:**
```prisma
model UserSession {
  id        Int      @id @default(autoincrement())
  userId    Int      @map("user_id")
  tokenJti  String   @unique @map("token_jti") @db.VarChar(255)
  expiresAt DateTime @map("expires_at") @db.Timestamptz
  ipAddress String?  @map("ip_address") @db.Inet
  userAgent String?  @map("user_agent") @db.Text
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz
  
  // Relations
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("user_sessions")
  @@index([tokenJti])
  @@index([userId])
  @@index([expiresAt])
  @@index([createdAt])
}
```

### 3. Audit Logs Table

```sql
CREATE TABLE audit_logs (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(100) NOT NULL,
    resource VARCHAR(100),
    resource_id VARCHAR(100),
    ip_address INET,
    user_agent TEXT,
    request_data JSONB,
    response_status INTEGER,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for audit querying
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
CREATE INDEX idx_audit_logs_composite ON audit_logs(user_id, action, created_at);
```

**Prisma Model:**
```prisma
model AuditLog {
  id             Int      @id @default(autoincrement())
  userId         Int?     @map("user_id")
  action         String   @db.VarChar(100)
  resource       String?  @db.VarChar(100)
  resourceId     String?  @map("resource_id") @db.VarChar(100)
  ipAddress      String?  @map("ip_address") @db.Inet
  userAgent      String?  @map("user_agent") @db.Text
  requestData    Json?    @map("request_data") @db.JsonB
  responseStatus Int?     @map("response_status")
  createdAt      DateTime @default(now()) @map("created_at") @db.Timestamptz
  
  // Relations
  user User? @relation(fields: [userId], references: [id], onDelete: SetNull)
  
  @@map("audit_logs")
  @@index([userId])
  @@index([action])
  @@index([resource])
  @@index([createdAt])
  @@index([userId, action, createdAt])
}
```

## Database Integration Patterns

### 1. Repository Pattern Implementation

```typescript
// Base Repository Interface
interface BaseRepository<T, CreateInput, UpdateInput> {
  findById(id: number): Promise<T | null>;
  findMany(filter?: any): Promise<T[]>;
  create(data: CreateInput): Promise<T>;
  update(id: number, data: UpdateInput): Promise<T>;
  delete(id: number): Promise<void>;
}

// User Repository
class UserRepository implements BaseRepository<User, CreateUserInput, UpdateUserInput> {
  constructor(private prisma: PrismaClient) {}

  async findById(id: number): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { id },
      select: {
        id: true,
        username: true,
        email: true,
        isActive: true,
        createdAt: true,
        updatedAt: true,
      }
    });
  }

  async findByUsername(username: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { username: username.toLowerCase() }
    });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { email: email.toLowerCase() }
    });
  }

  async create(data: CreateUserInput): Promise<User> {
    return this.prisma.user.create({
      data: {
        username: data.username.toLowerCase(),
        passwordHash: data.passwordHash,
        email: data.email?.toLowerCase(),
      },
      select: {
        id: true,
        username: true,
        email: true,
        isActive: true,
        createdAt: true,
        updatedAt: true,
      }
    });
  }

  async update(id: number, data: UpdateUserInput): Promise<User> {
    return this.prisma.user.update({
      where: { id },
      data: {
        ...data,
        username: data.username?.toLowerCase(),
        email: data.email?.toLowerCase(),
        updatedAt: new Date(),
      },
      select: {
        id: true,
        username: true,
        email: true,
        isActive: true,
        createdAt: true,
        updatedAt: true,
      }
    });
  }

  async delete(id: number): Promise<void> {
    await this.prisma.user.delete({
      where: { id }
    });
  }

  async findActiveUsers(limit: number = 50, offset: number = 0): Promise<User[]> {
    return this.prisma.user.findMany({
      where: { isActive: true },
      take: limit,
      skip: offset,
      orderBy: { createdAt: 'desc' },
      select: {
        id: true,
        username: true,
        email: true,
        isActive: true,
        createdAt: true,
        updatedAt: true,
      }
    });
  }
}
```

### 2. Session Management Repository

```typescript
class SessionRepository {
  constructor(private prisma: PrismaClient) {}

  async createSession(userId: number, tokenJti: string, expiresAt: Date, ipAddress?: string, userAgent?: string): Promise<UserSession> {
    return this.prisma.userSession.create({
      data: {
        userId,
        tokenJti,
        expiresAt,
        ipAddress,
        userAgent,
      }
    });
  }

  async findSessionByJti(tokenJti: string): Promise<UserSession | null> {
    return this.prisma.userSession.findUnique({
      where: { tokenJti },
      include: { user: true }
    });
  }

  async deleteSession(tokenJti: string): Promise<void> {
    await this.prisma.userSession.delete({
      where: { tokenJti }
    });
  }

  async deleteExpiredSessions(): Promise<number> {
    const result = await this.prisma.userSession.deleteMany({
      where: {
        expiresAt: {
          lt: new Date()
        }
      }
    });
    return result.count;
  }

  async deleteUserSessions(userId: number): Promise<number> {
    const result = await this.prisma.userSession.deleteMany({
      where: { userId }
    });
    return result.count;
  }
}
```

### 3. Audit Logging Repository

```typescript
class AuditLogRepository {
  constructor(private prisma: PrismaClient) {}

  async createAuditLog(data: CreateAuditLogInput): Promise<AuditLog> {
    return this.prisma.auditLog.create({
      data: {
        userId: data.userId,
        action: data.action,
        resource: data.resource,
        resourceId: data.resourceId,
        ipAddress: data.ipAddress,
        userAgent: data.userAgent,
        requestData: data.requestData,
        responseStatus: data.responseStatus,
      }
    });
  }

  async findUserAuditLogs(userId: number, limit: number = 100, offset: number = 0): Promise<AuditLog[]> {
    return this.prisma.auditLog.findMany({
      where: { userId },
      take: limit,
      skip: offset,
      orderBy: { createdAt: 'desc' }
    });
  }

  async findAuditLogsByAction(action: string, limit: number = 100): Promise<AuditLog[]> {
    return this.prisma.auditLog.findMany({
      where: { action },
      take: limit,
      orderBy: { createdAt: 'desc' },
      include: { user: true }
    });
  }

  async deleteOldAuditLogs(daysOld: number): Promise<number> {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - daysOld);
    
    const result = await this.prisma.auditLog.deleteMany({
      where: {
        createdAt: {
          lt: cutoffDate
        }
      }
    });
    return result.count;
  }
}
```

## Migration Management

### Initial Migration
```sql
-- migration: 001_initial_schema.sql
BEGIN;

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- User sessions table
CREATE TABLE user_sessions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_jti VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Audit logs table
CREATE TABLE audit_logs (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(100) NOT NULL,
    resource VARCHAR(100),
    resource_id VARCHAR(100),
    ip_address INET,
    user_agent TEXT,
    request_data JSONB,
    response_status INTEGER,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email) WHERE email IS NOT NULL;
CREATE INDEX idx_users_active ON users(is_active) WHERE is_active = true;
CREATE INDEX idx_users_created_at ON users(created_at);

CREATE INDEX idx_user_sessions_jti ON user_sessions(token_jti);
CREATE INDEX idx_user_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_user_sessions_expires_at ON user_sessions(expires_at);

CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);

-- Create updated_at trigger
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

COMMIT;
```

### Seed Data
```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcrypt';

const prisma = new PrismaClient();

async function main() {
  // Create admin user
  const adminPasswordHash = await bcrypt.hash('admin123', 12);
  
  const adminUser = await prisma.user.upsert({
    where: { username: 'admin' },
    update: {},
    create: {
      username: 'admin',
      passwordHash: adminPasswordHash,
      email: 'admin@example.com',
      isActive: true,
    },
  });

  // Create test user
  const testPasswordHash = await bcrypt.hash('test123', 12);
  
  const testUser = await prisma.user.upsert({
    where: { username: 'testuser' },
    update: {},
    create: {
      username: 'testuser',
      passwordHash: testPasswordHash,
      email: 'test@example.com',
      isActive: true,
    },
  });

  console.log('Database seeded successfully', { adminUser, testUser });
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

## Database Service Layer

```typescript
// database.service.ts
import { PrismaClient } from '@prisma/client';
import { UserRepository } from './repositories/user.repository';
import { SessionRepository } from './repositories/session.repository';
import { AuditLogRepository } from './repositories/audit-log.repository';

export class DatabaseService {
  private static instance: DatabaseService;
  private prisma: PrismaClient;
  
  public users: UserRepository;
  public sessions: SessionRepository;
  public auditLogs: AuditLogRepository;

  private constructor() {
    this.prisma = new PrismaClient({
      log: process.env.NODE_ENV === 'development' ? ['query', 'info', 'warn', 'error'] : ['error'],
    });

    this.users = new UserRepository(this.prisma);
    this.sessions = new SessionRepository(this.prisma);
    this.auditLogs = new AuditLogRepository(this.prisma);
  }

  public static getInstance(): DatabaseService {
    if (!DatabaseService.instance) {
      DatabaseService.instance = new DatabaseService();
    }
    return DatabaseService.instance;
  }

  public async connect(): Promise<void> {
    try {
      await this.prisma.$connect();
      console.log('Database connected successfully');
    } catch (error) {
      console.error('Database connection failed:', error);
      throw error;
    }
  }

  public async disconnect(): Promise<void> {
    await this.prisma.$disconnect();
  }

  public async healthCheck(): Promise<boolean> {
    try {
      await this.prisma.$queryRaw`SELECT 1`;
      return true;
    } catch (error) {
      console.error('Database health check failed:', error);
      return false;
    }
  }

  public getPrismaClient(): PrismaClient {
    return this.prisma;
  }
}
```

## Connection Management

```typescript
// connection.config.ts
export const databaseConfig = {
  url: process.env.DATABASE_URL!,
  pool: {
    min: parseInt(process.env.DB_POOL_MIN || '2'),
    max: parseInt(process.env.DB_POOL_MAX || '20'),
    timeout: parseInt(process.env.DB_POOL_TIMEOUT || '30000'),
    idleTimeout: parseInt(process.env.DB_POOL_IDLE_TIMEOUT || '600000'),
  },
  logging: process.env.NODE_ENV === 'development',
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
};
```

## Performance Optimization

### Query Optimization
```typescript
// Efficient user lookup with sessions
async findUserWithActiveSessions(userId: number): Promise<User & { sessions: UserSession[] }> {
  return this.prisma.user.findUnique({
    where: { id: userId },
    include: {
      sessions: {
        where: {
          expiresAt: {
            gt: new Date()
          }
        },
        orderBy: {
          createdAt: 'desc'
        }
      }
    }
  });
}

// Paginated user listing with search
async findUsersWithPagination(
  page: number = 1,
  limit: number = 20,
  search?: string
): Promise<{ users: User[]; total: number }> {
  const skip = (page - 1) * limit;
  const where = search
    ? {
        OR: [
          { username: { contains: search, mode: 'insensitive' as const } },
          { email: { contains: search, mode: 'insensitive' as const } }
        ]
      }
    : {};

  const [users, total] = await Promise.all([
    this.prisma.user.findMany({
      where,
      skip,
      take: limit,
      orderBy: { createdAt: 'desc' },
      select: {
        id: true,
        username: true,
        email: true,
        isActive: true,
        createdAt: true,
        updatedAt: true,
      }
    }),
    this.prisma.user.count({ where })
  ]);

  return { users, total };
}
```

### Cleanup Tasks
```typescript
// Scheduled cleanup service
export class DatabaseCleanupService {
  constructor(private db: DatabaseService) {}

  async cleanupExpiredSessions(): Promise<number> {
    return this.db.sessions.deleteExpiredSessions();
  }

  async cleanupOldAuditLogs(daysOld: number = 90): Promise<number> {
    return this.db.auditLogs.deleteOldAuditLogs(daysOld);
  }

  async runDailyCleanup(): Promise<void> {
    console.log('Starting daily database cleanup...');
    
    const expiredSessions = await this.cleanupExpiredSessions();
    const oldAuditLogs = await this.cleanupOldAuditLogs();
    
    console.log(`Cleanup completed: ${expiredSessions} sessions, ${oldAuditLogs} audit logs removed`);
  }
}
```

This database schema provides a solid foundation for the SaaS boilerplate with proper indexing, relationships, and integration patterns that support scalable user authentication and comprehensive audit logging.