# API Endpoints Specification

## Base Configuration

```
Base URL: http://localhost:3000/api
Content-Type: application/json
Authentication: Bearer {jwt_token}
```

## Authentication Endpoints

### POST /auth/register
Register a new user account.

**Request Body:**
```json
{
  "username": "string", // Required, 3-50 characters, alphanumeric + underscore
  "password": "string", // Required, min 8 characters
  "email": "string"     // Optional, valid email format
}
```

**Success Response (201):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": 1,
      "username": "johndoe",
      "email": "john@example.com",
      "isActive": true,
      "createdAt": "2024-01-15T10:30:00.000Z"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresAt": "2024-01-16T10:30:00.000Z"
  }
}
```

**Error Responses:**
```json
// 400 - Validation Error
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": {
      "username": ["Username is required"],
      "password": ["Password must be at least 8 characters"]
    }
  }
}

// 409 - User Already Exists
{
  "success": false,
  "error": {
    "code": "USER_EXISTS",
    "message": "Username already exists"
  }
}
```

### POST /auth/login
Authenticate user and return JWT token.

**Request Body:**
```json
{
  "username": "string", // Required
  "password": "string"  // Required
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": 1,
      "username": "johndoe",
      "email": "john@example.com",
      "isActive": true,
      "createdAt": "2024-01-15T10:30:00.000Z"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresAt": "2024-01-16T10:30:00.000Z"
  }
}
```

**Error Responses:**
```json
// 401 - Invalid Credentials
{
  "success": false,
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Invalid username or password"
  }
}

// 403 - Account Inactive
{
  "success": false,
  "error": {
    "code": "ACCOUNT_INACTIVE",
    "message": "Account has been deactivated"
  }
}
```

### POST /auth/logout
Invalidate current JWT token.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Success Response (200):**
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

### POST /auth/refresh
Refresh JWT token (extend expiration).

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresAt": "2024-01-16T10:30:00.000Z"
  }
}
```

### GET /auth/me
Get current authenticated user profile.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": 1,
      "username": "johndoe",
      "email": "john@example.com",
      "isActive": true,
      "createdAt": "2024-01-15T10:30:00.000Z",
      "updatedAt": "2024-01-15T10:30:00.000Z"
    }
  }
}
```

## User Management Endpoints

### GET /users
Get all users (admin functionality - future enhancement).

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Query Parameters:**
```
?page=1&limit=20&search=username&active=true
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "users": [
      {
        "id": 1,
        "username": "johndoe",
        "email": "john@example.com",
        "isActive": true,
        "createdAt": "2024-01-15T10:30:00.000Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 1,
      "totalPages": 1
    }
  }
}
```

### GET /users/:id
Get specific user by ID.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": 1,
      "username": "johndoe",
      "email": "john@example.com",
      "isActive": true,
      "createdAt": "2024-01-15T10:30:00.000Z",
      "updatedAt": "2024-01-15T10:30:00.000Z"
    }
  }
}
```

**Error Response (404):**
```json
{
  "success": false,
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User not found"
  }
}
```

### PUT /users/:id
Update user information.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Request Body:**
```json
{
  "username": "string", // Optional, 3-50 characters
  "email": "string",    // Optional, valid email format
  "password": "string"  // Optional, min 8 characters
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": 1,
      "username": "johndoe_updated",
      "email": "john.updated@example.com",
      "isActive": true,
      "createdAt": "2024-01-15T10:30:00.000Z",
      "updatedAt": "2024-01-15T11:30:00.000Z"
    }
  }
}
```

### DELETE /users/:id
Delete user account.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Success Response (200):**
```json
{
  "success": true,
  "message": "User deleted successfully"
}
```

### PATCH /users/:id/status
Activate or deactivate user account.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Request Body:**
```json
{
  "isActive": true // or false
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": 1,
      "username": "johndoe",
      "email": "john@example.com",
      "isActive": false,
      "createdAt": "2024-01-15T10:30:00.000Z",
      "updatedAt": "2024-01-15T11:30:00.000Z"
    }
  }
}
```

## Static File Serving Endpoints

### GET /
Root route that serves the main HTML file for the web application.

**Success Response (200):**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Infinity Server</title>
    <link rel="stylesheet" href="/static/css/main.css">
</head>
<body>
    <div id="app"></div>
    <script src="/static/js/main.js"></script>
</body>
</html>
```

### GET /static/*
Serve static assets (CSS, JavaScript, images, fonts, etc.) with proper caching headers.

**Examples:**
```
GET /static/css/main.css        # CSS files
GET /static/js/main.js          # JavaScript files  
GET /static/images/logo.png     # Image files
GET /static/assets/fonts/icon.woff  # Font files
```

**Success Response (200):**
- Content served with appropriate MIME type
- Cache headers: `Cache-Control: public, max-age=86400`
- ETag header for client-side caching
- Gzip compression for text-based files

**Error Response (404):**
```json
{
  "success": false,
  "error": {
    "code": "FILE_NOT_FOUND",
    "message": "Static file not found"
  }
}
```

### GET /{spa-route}
SPA (Single Page Application) fallback route. Any route that doesn't match `/api/*` or `/static/*` will serve the main `index.html` file to support client-side routing.

**Examples:**
```
GET /dashboard     # Serves index.html
GET /login        # Serves index.html  
GET /profile      # Serves index.html
GET /settings     # Serves index.html
```

**Success Response (200):**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Infinity Server</title>
    <link rel="stylesheet" href="/static/css/main.css">
</head>
<body>
    <div id="app"></div>
    <script src="/static/js/main.js"></script>
</body>
</html>
```

## System Endpoints

### GET /api/health
Health check endpoint for monitoring.

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "status": "healthy",
    "timestamp": "2024-01-15T10:30:00.000Z",
    "uptime": 3600,
    "services": {
      "database": "connected",
      "redis": "connected"
    }
  }
}
```

**Error Response (503):**
```json
{
  "success": false,
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Database connection failed",
    "details": {
      "database": "disconnected",
      "redis": "connected"
    }
  }
}
```

### GET /api/assets
Get information about available static assets (development/debugging endpoint).

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "css": [
      "/static/css/main.css",
      "/static/css/components.css"
    ],
    "js": [
      "/static/js/main.js",
      "/static/js/auth.js",
      "/static/js/api.js"
    ],
    "images": [
      "/static/images/logo.png",
      "/static/images/favicon.ico"
    ],
    "totalSize": "245KB",
    "lastModified": "2024-01-15T10:30:00.000Z"
  }
}
```

### GET /api/version
API version information.

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "version": "1.0.0",
    "apiVersion": "v1",
    "buildDate": "2024-01-15T10:30:00.000Z",
    "environment": "production"
  }
}
```

## Error Response Format

All error responses follow this standard format:

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable error message",
    "details": {} // Optional additional error details
  }
}
```

### Common Error Codes

| Code | Status | Description |
|------|--------|-------------|
| VALIDATION_ERROR | 400 | Request validation failed |
| UNAUTHORIZED | 401 | Missing or invalid authentication |
| FORBIDDEN | 403 | Insufficient permissions |
| NOT_FOUND | 404 | Resource not found |
| USER_EXISTS | 409 | Username/email already exists |
| INVALID_CREDENTIALS | 401 | Wrong username/password |
| ACCOUNT_INACTIVE | 403 | User account is deactivated |
| TOKEN_EXPIRED | 401 | JWT token has expired |
| TOKEN_INVALID | 401 | JWT token is malformed or invalid |
| RATE_LIMIT_EXCEEDED | 429 | Too many requests |
| INTERNAL_ERROR | 500 | Server error |
| SERVICE_UNAVAILABLE | 503 | External service unavailable |

## Rate Limiting

Rate limiting is applied to all endpoints:

```
General endpoints: 100 requests per 15 minutes per IP
Auth endpoints: 5 requests per 15 minutes per IP
```

Rate limit headers are included in responses:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 99
X-RateLimit-Reset: 1642248600
```

## Request/Response Examples

### Complete Registration Flow

```bash
# 1. Register new user
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser",
    "password": "securepassword123",
    "email": "test@example.com"
  }'

# 2. Login with credentials
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser",
    "password": "securepassword123"
  }'

# 3. Access protected endpoint
curl -X GET http://localhost:3000/api/auth/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# 4. Update user profile
curl -X PUT http://localhost:3000/api/users/1 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "email": "newemail@example.com"
  }'

# 5. Logout
curl -X POST http://localhost:3000/api/auth/logout \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

## Validation Rules

### Username
- Required for registration and login
- 3-50 characters
- Alphanumeric characters and underscores only
- Case-insensitive uniqueness

### Password
- Required for registration and login
- Minimum 8 characters
- No maximum length limit
- Hashed using bcrypt with salt rounds = 12

### Email
- Optional for registration
- Valid email format required when provided
- Case-insensitive uniqueness

## Security Headers

All responses include security headers:

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'
```

## CORS Configuration

```javascript
{
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}
```