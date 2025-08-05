# Testing Strategy & Implementation

## Overview

Comprehensive testing strategy for the SaaS boilerplate API server covering unit tests, integration tests, and end-to-end tests. Built with Jest, Supertest, and dedicated test database for reliable, isolated testing.

## Testing Philosophy

- **Test Pyramid**: More unit tests, fewer integration tests, minimal E2E tests
- **Isolated Testing**: Each test runs independently without side effects
- **Database Isolation**: Separate test database with transaction rollback
- **Realistic Data**: Use factories and fixtures for consistent test data
- **Fast Feedback**: Tests run quickly in development and CI/CD

## Testing Stack

```
├── Unit Testing: Jest + TypeScript
├── HTTP Testing: Supertest
├── Database Testing: Prisma + Test Database
├── Mocking: Jest mocks + manual mocks
├── Coverage: NYC/Istanbul
├── Test Data: Factory functions
└── CI/CD: GitHub Actions
```

## Test Structure

```
tests/
├── unit/                    # Unit tests (isolated components)
│   ├── controllers/
│   │   ├── auth.controller.test.ts
│   │   ├── user.controller.test.ts
│   │   └── health.controller.test.ts
│   ├── services/
│   │   ├── auth.service.test.ts
│   │   ├── user.service.test.ts
│   │   └── token.service.test.ts
│   ├── middleware/
│   │   ├── auth.middleware.test.ts
│   │   ├── validation.middleware.test.ts
│   │   └── error.middleware.test.ts
│   └── utils/
│       ├── logger.test.ts
│       └── validator.test.ts
├── integration/             # Integration tests (API endpoints)
│   ├── auth.integration.test.ts
│   ├── users.integration.test.ts
│   └── health.integration.test.ts
├── e2e/                     # End-to-end tests (complete workflows)
│   ├── user-registration.e2e.test.ts
│   ├── authentication-flow.e2e.test.ts
│   └── user-management.e2e.test.ts
├── fixtures/                # Test data and fixtures
│   ├── users.fixture.ts
│   ├── tokens.fixture.ts
│   └── requests.fixture.ts
├── helpers/                 # Test utilities
│   ├── database.helper.ts
│   ├── auth.helper.ts
│   └── request.helper.ts
└── setup/                   # Test configuration
    ├── jest.setup.ts
    ├── database.setup.ts
    └── teardown.ts
```

## Jest Configuration

```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: [
    '**/tests/**/*.test.ts',
    '**/tests/**/*.spec.ts'
  ],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/types/**/*.ts',
    '!src/generated/**/*.ts'
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  setupFilesAfterEnv: ['<rootDir>/tests/setup/jest.setup.ts'],
  globalTeardown: '<rootDir>/tests/setup/teardown.ts',
  testTimeout: 30000,
  maxWorkers: 1, // Ensure sequential test execution for database tests
  verbose: true
};
```

## Test Database Setup

```typescript
// tests/setup/database.setup.ts
import { PrismaClient } from '@prisma/client';
import { execSync } from 'child_process';

export class TestDatabaseSetup {
  private static prisma: PrismaClient;

  static async setup(): Promise<void> {
    // Ensure test database exists
    const testDatabaseUrl = process.env.TEST_DATABASE_URL;
    if (!testDatabaseUrl) {
      throw new Error('TEST_DATABASE_URL is not defined');
    }

    // Reset database schema
    execSync('npx prisma migrate reset --force --skip-seed', {
      env: { ...process.env, DATABASE_URL: testDatabaseUrl }
    });

    // Apply migrations
    execSync('npx prisma migrate deploy', {
      env: { ...process.env, DATABASE_URL: testDatabaseUrl }
    });

    // Initialize Prisma client
    this.prisma = new PrismaClient({
      datasources: {
        db: {
          url: testDatabaseUrl
        }
      }
    });

    await this.prisma.$connect();
  }

  static async cleanup(): Promise<void> {
    if (this.prisma) {
      await this.prisma.$disconnect();
    }
  }

  static getPrismaClient(): PrismaClient {
    return this.prisma;
  }

  static async clearDatabase(): Promise<void> {
    // Clear all tables in reverse dependency order
    await this.prisma.auditLog.deleteMany();
    await this.prisma.userSession.deleteMany();
    await this.prisma.user.deleteMany();
  }
}
```

```typescript
// tests/setup/jest.setup.ts
import { TestDatabaseSetup } from './database.setup';

beforeAll(async () => {
  await TestDatabaseSetup.setup();
});

afterAll(async () => {
  await TestDatabaseSetup.cleanup();
});

beforeEach(async () => {
  await TestDatabaseSetup.clearDatabase();
});
```

## Test Factories

```typescript
// tests/fixtures/users.fixture.ts
import bcrypt from 'bcrypt';
import { faker } from '@faker-js/faker';

export interface UserFixture {
  username: string;
  password: string;
  passwordHash: string;
  email?: string;
  isActive?: boolean;
}

export class UserFactory {
  static async create(overrides: Partial<UserFixture> = {}): Promise<UserFixture> {
    const password = overrides.password || 'password123';
    const passwordHash = await bcrypt.hash(password, 12);

    return {
      username: faker.internet.userName().toLowerCase(),
      password,
      passwordHash,
      email: faker.internet.email().toLowerCase(),
      isActive: true,
      ...overrides
    };
  }

  static async createMany(count: number, overrides: Partial<UserFixture> = {}): Promise<UserFixture[]> {
    const users: UserFixture[] = [];
    for (let i = 0; i < count; i++) {
      users.push(await this.create(overrides));
    }
    return users;
  }

  static async createInactive(): Promise<UserFixture> {
    return this.create({ isActive: false });
  }

  static async createWithoutEmail(): Promise<UserFixture> {
    return this.create({ email: undefined });
  }
}
```

```typescript
// tests/fixtures/tokens.fixture.ts
import jwt from 'jsonwebtoken';
import { v4 as uuidv4 } from 'uuid';

export interface TokenFixture {
  token: string;
  payload: jwt.JwtPayload;
  jti: string;
}

export class TokenFactory {
  static create(userId: number, username: string, expiresIn: string = '1h'): TokenFixture {
    const jti = uuidv4();
    const payload = {
      sub: userId.toString(),
      username,
      jti,
    };

    const token = jwt.sign(payload, process.env.JWT_SECRET!, {
      expiresIn,
      issuer: process.env.JWT_ISSUER,
      audience: process.env.JWT_AUDIENCE,
    });

    return { token, payload: payload as jwt.JwtPayload, jti };
  }

  static createExpired(userId: number, username: string): TokenFixture {
    return this.create(userId, username, '-1h'); // Already expired
  }

  static createInvalid(): string {
    return 'invalid.jwt.token';
  }
}
```

## Unit Tests

### Controller Unit Tests
```typescript
// tests/unit/controllers/auth.controller.test.ts
import { Request, Response } from 'express';
import { AuthController } from '../../../src/controllers/auth.controller';
import { AuthService } from '../../../src/services/auth.service';
import { UserFactory } from '../../fixtures/users.fixture';

// Mock the auth service
jest.mock('../../../src/services/auth.service');
const MockedAuthService = AuthService as jest.MockedClass<typeof AuthService>;

describe('AuthController', () => {
  let authController: AuthController;
  let mockAuthService: jest.Mocked<AuthService>;
  let mockRequest: Partial<Request>;
  let mockResponse: Partial<Response>;

  beforeEach(() => {
    mockAuthService = new MockedAuthService() as jest.Mocked<AuthService>;
    authController = new AuthController(mockAuthService);
    
    mockRequest = {
      body: {},
      ip: '127.0.0.1',
      get: jest.fn(),
    };
    
    mockResponse = {
      status: jest.fn().mockReturnThis(),
      json: jest.fn().mockReturnThis(),
    };
  });

  describe('register', () => {
    it('should register a new user successfully', async () => {
      // Arrange
      const userFixture = await UserFactory.create();
      const registrationData = {
        username: userFixture.username,
        password: userFixture.password,
        email: userFixture.email,
      };
      
      const expectedResponse = {
        user: {
          id: 1,
          username: userFixture.username,
          email: userFixture.email,
          isActive: true,
          createdAt: new Date().toISOString(),
        },
        token: 'jwt-token',
        expiresAt: new Date().toISOString(),
      };

      mockRequest.body = registrationData;
      mockAuthService.register.mockResolvedValue(expectedResponse);

      // Act
      await authController.register(mockRequest as Request, mockResponse as Response);

      // Assert
      expect(mockAuthService.register).toHaveBeenCalledWith(
        registrationData,
        '127.0.0.1',
        undefined
      );
      expect(mockResponse.status).toHaveBeenCalledWith(201);
      expect(mockResponse.json).toHaveBeenCalledWith({
        success: true,
        data: expectedResponse,
      });
    });

    it('should return validation error for invalid input', async () => {
      // Arrange
      mockRequest.body = { username: 'ab' }; // Too short
      const error = new Error('Validation failed');
      error.name = 'ValidationError';
      mockAuthService.register.mockRejectedValue(error);

      // Act
      await authController.register(mockRequest as Request, mockResponse as Response);

      // Assert
      expect(mockResponse.status).toHaveBeenCalledWith(400);
      expect(mockResponse.json).toHaveBeenCalledWith({
        success: false,
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Validation failed',
        },
      });
    });
  });

  describe('login', () => {
    it('should login user successfully', async () => {
      // Arrange
      const userFixture = await UserFactory.create();
      const loginData = {
        username: userFixture.username,
        password: userFixture.password,
      };

      const expectedResponse = {
        user: {
          id: 1,
          username: userFixture.username,
          email: userFixture.email,
          isActive: true,
          createdAt: new Date().toISOString(),
        },
        token: 'jwt-token',
        expiresAt: new Date().toISOString(),
      };

      mockRequest.body = loginData;
      mockAuthService.login.mockResolvedValue(expectedResponse);

      // Act
      await authController.login(mockRequest as Request, mockResponse as Response);

      // Assert
      expect(mockAuthService.login).toHaveBeenCalledWith(
        loginData,
        '127.0.0.1',
        undefined
      );
      expect(mockResponse.status).toHaveBeenCalledWith(200);
      expect(mockResponse.json).toHaveBeenCalledWith({
        success: true,
        data: expectedResponse,
      });
    });

    it('should return error for invalid credentials', async () => {
      // Arrange
      mockRequest.body = {
        username: 'nonexistent',
        password: 'wrongpassword',
      };
      
      const error = new Error('Invalid credentials');
      error.name = 'AuthenticationError';
      mockAuthService.login.mockRejectedValue(error);

      // Act
      await authController.login(mockRequest as Request, mockResponse as Response);

      // Assert
      expect(mockResponse.status).toHaveBeenCalledWith(401);
      expect(mockResponse.json).toHaveBeenCalledWith({
        success: false,
        error: {
          code: 'INVALID_CREDENTIALS',
          message: 'Invalid credentials',
        },
      });
    });
  });
});
```

### Service Unit Tests
```typescript
// tests/unit/services/auth.service.test.ts
import bcrypt from 'bcrypt';
import { AuthService } from '../../../src/services/auth.service';
import { UserRepository } from '../../../src/repositories/user.repository';
import { SessionRepository } from '../../../src/repositories/session.repository';
import { TokenService } from '../../../src/services/token.service';
import { UserFactory } from '../../fixtures/users.fixture';

// Mock dependencies
jest.mock('../../../src/repositories/user.repository');
jest.mock('../../../src/repositories/session.repository');
jest.mock('../../../src/services/token.service');
jest.mock('bcrypt');

const MockedUserRepository = UserRepository as jest.MockedClass<typeof UserRepository>;
const MockedSessionRepository = SessionRepository as jest.MockedClass<typeof SessionRepository>;
const MockedTokenService = TokenService as jest.MockedClass<typeof TokenService>;
const mockedBcrypt = bcrypt as jest.Mocked<typeof bcrypt>;

describe('AuthService', () => {
  let authService: AuthService;
  let mockUserRepository: jest.Mocked<UserRepository>;
  let mockSessionRepository: jest.Mocked<SessionRepository>;
  let mockTokenService: jest.Mocked<TokenService>;

  beforeEach(() => {
    mockUserRepository = new MockedUserRepository() as jest.Mocked<UserRepository>;
    mockSessionRepository = new MockedSessionRepository() as jest.Mocked<SessionRepository>;
    mockTokenService = new MockedTokenService() as jest.Mocked<TokenService>;
    
    authService = new AuthService(
      mockUserRepository,
      mockSessionRepository,
      mockTokenService
    );
  });

  describe('register', () => {
    it('should register a new user successfully', async () => {
      // Arrange
      const userFixture = await UserFactory.create();
      const registrationData = {
        username: userFixture.username,
        password: userFixture.password,
        email: userFixture.email,
      };

      mockUserRepository.findByUsername.mockResolvedValue(null);
      mockUserRepository.findByEmail.mockResolvedValue(null);
      mockedBcrypt.hash.mockResolvedValue(userFixture.passwordHash as never);
      
      const createdUser = {
        id: 1,
        username: userFixture.username,
        email: userFixture.email,
        isActive: true,
        createdAt: new Date(),
        updatedAt: new Date(),
      };
      
      mockUserRepository.create.mockResolvedValue(createdUser);
      mockTokenService.generateToken.mockReturnValue({
        token: 'jwt-token',
        expiresAt: new Date(),
        jti: 'jti-123',
      });
      mockSessionRepository.createSession.mockResolvedValue({} as any);

      // Act
      const result = await authService.register(registrationData, '127.0.0.1');

      // Assert
      expect(mockUserRepository.findByUsername).toHaveBeenCalledWith(userFixture.username);
      expect(mockUserRepository.findByEmail).toHaveBeenCalledWith(userFixture.email);
      expect(mockedBcrypt.hash).toHaveBeenCalledWith(userFixture.password, 12);
      expect(mockUserRepository.create).toHaveBeenCalledWith({
        username: userFixture.username,
        passwordHash: userFixture.passwordHash,
        email: userFixture.email,
      });
      expect(result.user).toEqual(createdUser);
      expect(result.token).toBe('jwt-token');
    });

    it('should throw error if username already exists', async () => {
      // Arrange
      const userFixture = await UserFactory.create();
      const registrationData = {
        username: userFixture.username,
        password: userFixture.password,
        email: userFixture.email,
      };

      mockUserRepository.findByUsername.mockResolvedValue({} as any);

      // Act & Assert
      await expect(authService.register(registrationData, '127.0.0.1'))
        .rejects.toThrow('Username already exists');
    });
  });

  describe('login', () => {
    it('should login user successfully', async () => {
      // Arrange
      const userFixture = await UserFactory.create();
      const loginData = {
        username: userFixture.username,
        password: userFixture.password,
      };

      const dbUser = {
        id: 1,
        username: userFixture.username,
        passwordHash: userFixture.passwordHash,
        email: userFixture.email,
        isActive: true,
        createdAt: new Date(),
        updatedAt: new Date(),
      };

      mockUserRepository.findByUsername.mockResolvedValue(dbUser);
      mockedBcrypt.compare.mockResolvedValue(true as never);
      mockTokenService.generateToken.mockReturnValue({
        token: 'jwt-token',
        expiresAt: new Date(),
        jti: 'jti-123',
      });
      mockSessionRepository.createSession.mockResolvedValue({} as any);

      // Act
      const result = await authService.login(loginData, '127.0.0.1');

      // Assert
      expect(mockUserRepository.findByUsername).toHaveBeenCalledWith(userFixture.username);
      expect(mockedBcrypt.compare).toHaveBeenCalledWith(userFixture.password, userFixture.passwordHash);
      expect(result.user.id).toBe(1);
      expect(result.token).toBe('jwt-token');
    });

    it('should throw error for invalid credentials', async () => {
      // Arrange
      mockUserRepository.findByUsername.mockResolvedValue(null);

      // Act & Assert
      await expect(authService.login({ username: 'nonexistent', password: 'wrong' }, '127.0.0.1'))
        .rejects.toThrow('Invalid credentials');
    });
  });
});
```

## Integration Tests

```typescript
// tests/integration/auth.integration.test.ts
import request from 'supertest';
import { app } from '../../src/app';
import { TestDatabaseSetup } from '../setup/database.setup';
import { UserFactory } from '../fixtures/users.fixture';

describe('Auth Integration Tests', () => {
  let prisma: any;

  beforeAll(() => {
    prisma = TestDatabaseSetup.getPrismaClient();
  });

  describe('POST /api/auth/register', () => {
    it('should register a new user', async () => {
      // Arrange
      const userFixture = await UserFactory.create();
      const registrationData = {
        username: userFixture.username,
        password: userFixture.password,
        email: userFixture.email,
      };

      // Act
      const response = await request(app)
        .post('/api/auth/register')
        .send(registrationData)
        .expect(201);

      // Assert
      expect(response.body.success).toBe(true);
      expect(response.body.data.user.username).toBe(userFixture.username);
      expect(response.body.data.user.email).toBe(userFixture.email);
      expect(response.body.data.token).toBeDefined();
      expect(response.body.data.expiresAt).toBeDefined();

      // Verify user was created in database
      const dbUser = await prisma.user.findUnique({
        where: { username: userFixture.username }
      });
      expect(dbUser).toBeTruthy();
      expect(dbUser.isActive).toBe(true);
    });

    it('should return error for duplicate username', async () => {
      // Arrange
      const userFixture = await UserFactory.create();
      await prisma.user.create({
        data: {
          username: userFixture.username,
          passwordHash: userFixture.passwordHash,
          email: userFixture.email,
        }
      });

      // Act
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          username: userFixture.username,
          password: 'differentpassword',
          email: 'different@example.com',
        })
        .expect(409);

      // Assert
      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('USER_EXISTS');
    });
  });

  describe('POST /api/auth/login', () => {
    it('should login existing user', async () => {
      // Arrange
      const userFixture = await UserFactory.create();
      await prisma.user.create({
        data: {
          username: userFixture.username,
          passwordHash: userFixture.passwordHash,
          email: userFixture.email,
        }
      });

      // Act
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          username: userFixture.username,
          password: userFixture.password,
        })
        .expect(200);

      // Assert
      expect(response.body.success).toBe(true);
      expect(response.body.data.user.username).toBe(userFixture.username);
      expect(response.body.data.token).toBeDefined();

      // Verify session was created
      const session = await prisma.userSession.findFirst({
        where: { user: { username: userFixture.username } }
      });
      expect(session).toBeTruthy();
    });

    it('should return error for invalid credentials', async () => {
      // Act
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          username: 'nonexistent',
          password: 'wrongpassword',
        })
        .expect(401);

      // Assert
      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('INVALID_CREDENTIALS');
    });
  });

  describe('GET /api/auth/me', () => {
    it('should return current user profile', async () => {
      // Arrange
      const userFixture = await UserFactory.create();
      const user = await prisma.user.create({
        data: {
          username: userFixture.username,
          passwordHash: userFixture.passwordHash,
          email: userFixture.email,
        }
      });

      // Login to get token
      const loginResponse = await request(app)
        .post('/api/auth/login')
        .send({
          username: userFixture.username,
          password: userFixture.password,
        });

      const token = loginResponse.body.data.token;

      // Act
      const response = await request(app)
        .get('/api/auth/me')
        .set('Authorization', `Bearer ${token}`)
        .expect(200);

      // Assert
      expect(response.body.success).toBe(true);
      expect(response.body.data.user.id).toBe(user.id);
      expect(response.body.data.user.username).toBe(userFixture.username);
    });

    it('should return error for missing token', async () => {
      // Act
      const response = await request(app)
        .get('/api/auth/me')
        .expect(401);

      // Assert
      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('UNAUTHORIZED');
    });
  });
});
```

## E2E Tests

```typescript
// tests/e2e/authentication-flow.e2e.test.ts
import request from 'supertest';
import { app } from '../../src/app';
import { TestDatabaseSetup } from '../setup/database.setup';
import { UserFactory } from '../fixtures/users.fixture';

describe('Authentication Flow E2E', () => {
  let prisma: any;

  beforeAll(() => {
    prisma = TestDatabaseSetup.getPrismaClient();
  });

  it('should complete full authentication flow', async () => {
    const userFixture = await UserFactory.create();
    let token: string;
    let userId: number;

    // Step 1: Register new user
    const registerResponse = await request(app)
      .post('/api/auth/register')
      .send({
        username: userFixture.username,
        password: userFixture.password,
        email: userFixture.email,
      })
      .expect(201);

    expect(registerResponse.body.success).toBe(true);
    token = registerResponse.body.data.token;
    userId = registerResponse.body.data.user.id;

    // Step 2: Access protected endpoint
    const profileResponse = await request(app)
      .get('/api/auth/me')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);

    expect(profileResponse.body.data.user.id).toBe(userId);

    // Step 3: Update user profile
    const updateResponse = await request(app)
      .put(`/api/users/${userId}`)
      .set('Authorization', `Bearer ${token}`)
      .send({
        email: 'updated@example.com'
      })
      .expect(200);

    expect(updateResponse.body.data.user.email).toBe('updated@example.com');

    // Step 4: Logout
    await request(app)
      .post('/api/auth/logout')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);

    // Step 5: Verify token is invalidated
    await request(app)
      .get('/api/auth/me')
      .set('Authorization', `Bearer ${token}`)
      .expect(401);

    // Step 6: Login again with updated email
    const loginResponse = await request(app)
      .post('/api/auth/login')
      .send({
        username: userFixture.username,
        password: userFixture.password,
      })
      .expect(200);

    expect(loginResponse.body.data.user.email).toBe('updated@example.com');
  });
});
```

## Test Commands

```json
{
  "scripts": {
    "test": "jest",
    "test:unit": "jest tests/unit",
    "test:integration": "jest tests/integration",
    "test:e2e": "jest tests/e2e",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --watchAll=false"
  }
}
```

## CI/CD Pipeline

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: infinity_server_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Setup test database
        run: |
          npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/infinity_server_test
          
      - name: Run tests
        run: npm run test:ci
        env:
          NODE_ENV: test
          DATABASE_URL: postgresql://postgres:test@localhost:5432/infinity_server_test
          TEST_DATABASE_URL: postgresql://postgres:test@localhost:5432/infinity_server_test
          JWT_SECRET: test-secret-key
          
      - name: Upload coverage reports
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
```

This comprehensive testing strategy ensures reliable, maintainable code with excellent coverage and fast feedback loops for development.