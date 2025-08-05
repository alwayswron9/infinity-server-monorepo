# Monorepo Structure & Configuration

## Overview

The infinity-server monorepo combines API server functionality with static file serving in a unified TypeScript codebase. This structure allows for seamless full-stack development, shared utilities, and coordinated deployments.

## Workspace Configuration

### Root Package.json
```json
{
  "name": "infinity-server",
  "version": "1.0.0",
  "description": "A TypeScript monorepo for SaaS boilerplate with API server and static file serving",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "dev": "concurrently \"npm run dev:server\" \"npm run watch:static\"",
    "dev:server": "npm run dev --workspace=packages/server",
    "build": "npm run build --workspaces",
    "build:server": "npm run build --workspace=packages/server",
    "build:static": "npm run build:static --workspace=packages/server",
    "start": "npm run start --workspace=packages/server",
    "test": "npm run test --workspaces",
    "test:coverage": "npm run test:coverage --workspaces",
    "lint": "eslint packages/*/src/**/*.ts",
    "format": "prettier --write packages/*/src/**/*.{ts,js,json}",
    "type-check": "tsc --noEmit --composite false",
    "clean": "npm run clean --workspaces && rm -rf node_modules/.cache",
    "db:migrate": "npm run db:migrate --workspace=packages/server",
    "db:generate": "npm run db:generate --workspace=packages/server",
    "db:seed": "npm run db:seed --workspace=packages/server",
    "db:reset": "npm run db:reset --workspace=packages/server"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "concurrently": "^8.0.0",
    "eslint": "^8.0.0",
    "prettier": "^3.0.0",
    "typescript": "^5.0.0"
  },
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=8.0.0"
  }
}
```

### Root TypeScript Configuration
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "composite": true,
    "incremental": true,
    "baseUrl": ".",
    "paths": {
      "@shared/*": ["packages/shared/src/*"],
      "@server/*": ["packages/server/src/*"]
    }
  },
  "references": [
    { "path": "./packages/server" },
    { "path": "./packages/shared" }
  ],
  "exclude": ["node_modules", "dist", "coverage"]
}
```

## Package Structure

### Server Package (`packages/server/`)

#### Package.json
```json
{
  "name": "@infinity-server/server",
  "version": "1.0.0",
  "description": "API server with static file serving",
  "main": "dist/app.js",
  "scripts": {
    "dev": "tsx watch src/app.ts",
    "build": "tsc && npm run build:static",
    "build:static": "node scripts/build-static.js",
    "start": "node dist/app.js",
    "test": "jest",
    "test:unit": "jest tests/unit",
    "test:integration": "jest tests/integration",
    "test:e2e": "jest tests/e2e",
    "test:coverage": "jest --coverage",
    "test:watch": "jest --watch",
    "lint": "eslint src/**/*.ts",
    "format": "prettier --write src/**/*.{ts,js,json}",
    "type-check": "tsc --noEmit",
    "db:migrate": "prisma migrate deploy",
    "db:generate": "prisma generate",
    "db:seed": "tsx prisma/seed.ts",
    "db:reset": "prisma migrate reset --force",
    "clean": "rm -rf dist coverage"
  },
  "dependencies": {
    "express": "^4.18.0",
    "bcrypt": "^5.1.0",
    "jsonwebtoken": "^9.0.0",
    "winston": "^3.10.0",
    "helmet": "^7.0.0",
    "cors": "^2.8.5",
    "compression": "^1.7.4",
    "express-rate-limit": "^6.10.0",
    "zod": "^3.22.0",
    "@prisma/client": "^5.0.0",
    "dotenv": "^16.3.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.0",
    "@types/bcrypt": "^5.0.0",
    "@types/jsonwebtoken": "^9.0.0",
    "@types/compression": "^1.7.0",
    "@types/cors": "^2.8.0",
    "@types/jest": "^29.5.0",
    "jest": "^29.6.0",
    "supertest": "^6.3.0",
    "@types/supertest": "^2.0.0",
    "ts-jest": "^29.1.0",
    "tsx": "^3.12.0",
    "prisma": "^5.0.0",
    "typescript": "^5.0.0"
  }
}
```

#### TypeScript Configuration
```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "composite": true,
    "tsBuildInfoFile": "./dist/.tsbuildinfo"
  },
  "include": [
    "src/**/*",
    "tests/**/*"
  ],
  "exclude": [
    "node_modules",
    "dist",
    "public"
  ],
  "references": [
    { "path": "../shared" }
  ]
}
```

#### Jest Configuration
```javascript
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
    '!src/types/**/*.ts'
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
  testTimeout: 30000,
  maxWorkers: 1
};
```

### Shared Package (`packages/shared/`)

#### Package.json
```json
{
  "name": "@infinity-server/shared",
  "version": "1.0.0",
  "description": "Shared utilities and types",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "clean": "rm -rf dist",
    "lint": "eslint src/**/*.ts",
    "format": "prettier --write src/**/*.{ts,js,json}",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "zod": "^3.22.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0"
  }
}
```

#### TypeScript Configuration
```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "composite": true,
    "tsBuildInfoFile": "./dist/.tsbuildinfo"
  },
  "include": [
    "src/**/*"
  ],
  "exclude": [
    "node_modules",
    "dist"
  ]
}
```

## Detailed Directory Structure

```
infinity-server/
├── .github/                       # GitHub Actions workflows
│   └── workflows/
│       ├── ci.yml                # Continuous integration
│       ├── deploy.yml            # Deployment pipeline
│       └── test.yml              # Test automation
├── packages/
│   ├── server/                   # Main server package
│   │   ├── src/
│   │   │   ├── controllers/      # HTTP request handlers
│   │   │   │   ├── auth.controller.ts
│   │   │   │   ├── user.controller.ts
│   │   │   │   ├── static.controller.ts
│   │   │   │   └── health.controller.ts
│   │   │   ├── middleware/       # Express middleware
│   │   │   │   ├── auth.middleware.ts
│   │   │   │   ├── logging.middleware.ts
│   │   │   │   ├── validation.middleware.ts
│   │   │   │   ├── static.middleware.ts
│   │   │   │   ├── compression.middleware.ts
│   │   │   │   └── error.middleware.ts
│   │   │   ├── services/         # Business logic layer
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── user.service.ts
│   │   │   │   ├── token.service.ts
│   │   │   │   ├── static.service.ts
│   │   │   │   └── audit.service.ts
│   │   │   ├── repositories/     # Data access layer
│   │   │   │   ├── user.repository.ts
│   │   │   │   ├── session.repository.ts
│   │   │   │   ├── audit-log.repository.ts
│   │   │   │   └── base.repository.ts
│   │   │   ├── routes/           # Route definitions
│   │   │   │   ├── api/          # API routes
│   │   │   │   │   ├── auth.routes.ts
│   │   │   │   │   ├── user.routes.ts
│   │   │   │   │   ├── health.routes.ts
│   │   │   │   │   └── index.ts
│   │   │   │   ├── static.routes.ts
│   │   │   │   └── index.ts
│   │   │   ├── utils/            # Utility functions
│   │   │   │   ├── logger.ts
│   │   │   │   ├── validator.ts
│   │   │   │   ├── constants.ts
│   │   │   │   ├── crypto.ts
│   │   │   │   └── response.ts
│   │   │   ├── config/           # Configuration management
│   │   │   │   ├── database.ts
│   │   │   │   ├── jwt.ts
│   │   │   │   ├── static.ts
│   │   │   │   ├── cors.ts
│   │   │   │   └── server.ts
│   │   │   ├── types/            # TypeScript type definitions
│   │   │   │   ├── auth.types.ts
│   │   │   │   ├── user.types.ts
│   │   │   │   ├── api.types.ts
│   │   │   │   ├── express.types.ts
│   │   │   │   └── index.ts
│   │   │   └── app.ts            # Express application setup
│   │   ├── public/               # Static files directory
│   │   │   ├── index.html        # Main HTML file
│   │   │   ├── css/
│   │   │   │   ├── main.css      # Main stylesheet
│   │   │   │   ├── components.css # Component styles
│   │   │   │   └── theme.css     # Theme variables
│   │   │   ├── js/
│   │   │   │   ├── main.js       # Main application logic
│   │   │   │   ├── auth.js       # Authentication handling
│   │   │   │   ├── api.js        # API client wrapper
│   │   │   │   ├── utils.js      # Client-side utilities
│   │   │   │   └── components.js # UI components
│   │   │   ├── images/
│   │   │   │   ├── logo.png
│   │   │   │   ├── favicon.ico
│   │   │   │   └── placeholder.png
│   │   │   └── assets/
│   │   │       ├── fonts/
│   │   │       │   ├── Inter-Variable.woff2
│   │   │       │   └── icons.woff2
│   │   │       └── icons/
│   │   │           ├── user.svg
│   │   │           ├── settings.svg
│   │   │           └── logout.svg
│   │   ├── tests/                # Test files
│   │   │   ├── unit/             # Unit tests
│   │   │   │   ├── controllers/
│   │   │   │   ├── services/
│   │   │   │   ├── middleware/
│   │   │   │   └── utils/
│   │   │   ├── integration/      # Integration tests
│   │   │   │   ├── auth.integration.test.ts
│   │   │   │   ├── users.integration.test.ts
│   │   │   │   └── static.integration.test.ts
│   │   │   ├── e2e/              # End-to-end tests
│   │   │   │   ├── authentication-flow.e2e.test.ts
│   │   │   │   ├── user-management.e2e.test.ts
│   │   │   │   └── static-serving.e2e.test.ts
│   │   │   ├── fixtures/         # Test data
│   │   │   │   ├── users.fixture.ts
│   │   │   │   ├── tokens.fixture.ts
│   │   │   │   └── requests.fixture.ts
│   │   │   ├── helpers/          # Test utilities
│   │   │   │   ├── database.helper.ts
│   │   │   │   ├── auth.helper.ts
│   │   │   │   └── request.helper.ts
│   │   │   └── setup/            # Test configuration
│   │   │       ├── jest.setup.ts
│   │   │       ├── database.setup.ts
│   │   │       └── teardown.ts
│   │   ├── prisma/               # Database schema and migrations
│   │   │   ├── schema.prisma     # Prisma schema definition
│   │   │   ├── migrations/       # Database migrations
│   │   │   │   ├── 20240101000000_init/
│   │   │   │   └── migration.sql
│   │   │   └── seed.ts           # Database seeding script
│   │   ├── scripts/              # Build and utility scripts
│   │   │   ├── build-static.js   # Static file build script
│   │   │   ├── cleanup.js        # Cleanup script
│   │   │   └── dev-setup.js      # Development setup
│   │   ├── logs/                 # Log files (development)
│   │   ├── coverage/             # Test coverage reports
│   │   ├── dist/                 # Compiled TypeScript output
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── jest.config.js
│   │   ├── .env.example
│   │   └── Dockerfile
│   └── shared/                   # Shared utilities package
│       ├── src/
│       │   ├── types/            # Shared type definitions
│       │   │   ├── api.types.ts
│       │   │   ├── auth.types.ts
│       │   │   ├── user.types.ts
│       │   │   └── index.ts
│       │   ├── utils/            # Shared utility functions
│       │   │   ├── validation.ts
│       │   │   ├── constants.ts
│       │   │   ├── formatters.ts
│       │   │   └── helpers.ts
│       │   ├── schemas/          # Zod validation schemas
│       │   │   ├── auth.schemas.ts
│       │   │   ├── user.schemas.ts
│       │   │   └── index.ts
│       │   └── index.ts          # Main export file
│       ├── package.json
│       ├── tsconfig.json
│       └── dist/                 # Compiled output
├── plan/                         # Project documentation
│   ├── server-specification.md
│   ├── api-endpoints.md
│   ├── database-schema.md
│   ├── testing-strategy.md
│   └── monorepo-structure.md     # This file
├── docs/                         # Additional documentation
│   ├── deployment.md
│   ├── development.md
│   └── api-guide.md
├── .env.example                  # Environment variables template
├── .gitignore                    # Git ignore rules
├── .eslintrc.js                  # ESLint configuration
├── .prettierrc                   # Prettier configuration
├── docker-compose.yml            # Docker Compose for development
├── Dockerfile                    # Production Docker image
├── package.json                  # Root package configuration
├── tsconfig.json                 # Root TypeScript configuration
└── README.md                     # Project documentation
```

## Build Process

### Development Mode
```bash
# Start both server and static file watching
npm run dev

# Or run components separately
npm run dev:server        # Start server in watch mode
npm run watch:static       # Watch static files for changes
```

### Production Build
```bash
# Build all packages
npm run build

# Build process:
# 1. Compile TypeScript in all packages
# 2. Optimize static files (minification, compression)
# 3. Generate Prisma client
# 4. Run tests and linting
```

## Environment Configuration

### Development (.env.development)
```env
NODE_ENV=development
PORT=3000
HOST=localhost

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/infinity_server_dev
TEST_DATABASE_URL=postgresql://user:password@localhost:5432/infinity_server_test

# JWT
JWT_SECRET=dev-secret-key
JWT_EXPIRES_IN=24h

# Static Files
STATIC_FILES_PATH=./public
ENABLE_STATIC_COMPRESSION=false
STATIC_MAX_AGE=0

# Logging
LOG_LEVEL=debug
LOG_TO_FILE=false
```

### Production (.env.production)
```env
NODE_ENV=production
PORT=3000
HOST=0.0.0.0

# Database
DATABASE_URL=postgresql://user:password@prod-db:5432/infinity_server

# JWT
JWT_SECRET=super-secure-production-secret
JWT_EXPIRES_IN=1h

# Static Files
STATIC_FILES_PATH=./public
ENABLE_STATIC_COMPRESSION=true
STATIC_MAX_AGE=86400

# Logging
LOG_LEVEL=info
LOG_TO_FILE=true
LOG_FILE_PATH=/var/log/infinity-server/app.log
```

## Docker Configuration

### Development Docker Compose
```yaml
version: '3.8'
services:
  app:
    build: 
      context: .
      dockerfile: packages/server/Dockerfile
      target: development
    ports:
      - "3000:3000"
    volumes:
      - ./packages/server/src:/app/src
      - ./packages/server/public:/app/public
      - ./packages/shared:/app/shared
    environment:
      - NODE_ENV=development
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: infinity_server_dev
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Production Dockerfile
```dockerfile
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./
COPY packages/server/package*.json ./packages/server/
COPY packages/shared/package*.json ./packages/shared/

FROM base AS dependencies
RUN npm ci --only=production && npm cache clean --force

FROM base AS build
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine AS production
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY --from=build /app/packages/server/dist ./dist
COPY --from=build /app/packages/server/public ./public
COPY --from=build /app/packages/shared/dist ./shared

EXPOSE 3000
CMD ["node", "dist/app.js"]
```

## Deployment Strategy

### CI/CD Pipeline
```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:coverage
      - run: npm run lint
      - run: npm run type-check

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: docker build -t infinity-server .
      - run: docker push registry/infinity-server:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: kubectl apply -f k8s/
      - run: kubectl rollout restart deployment/infinity-server
```

This monorepo structure provides a solid foundation for developing, testing, and deploying a full-stack TypeScript application with API server and static file serving capabilities.