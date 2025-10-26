# Movie API

A production-ready RESTful API for managing movie data with user authentication and authorization.

## Features

- **PostgreSQL Database** - Robust relational database with proper connection pooling
- **Versioned API** - API versioning support (v1)
- **Email Integration** - Mailtrap for email delivery (activation tokens, password resets)
- **Token-based Authentication** - Stateful authentication using bearer tokens
- **Permission-based Authorization** - Fine-grained permissions (read/write) for users
- **Fulltext Search** - PostgreSQL fulltext search on movie titles
- **Advanced Filtering** - Filter movies by genres with support for multiple filters
- **Pagination** - Efficient pagination with configurable page size and sorting
- **Optimistic Concurrency** - Handles data race conditions using version fields
- **Rate Limiting** - Per-IP token bucket rate limiter (default: 2 req/s with burst of 4)
- **CORS Support** - Configurable trusted origins with preflight request handling
- **Metrics** - Built-in metrics for monitoring (total requests, processing time, status codes)
- **Graceful Shutdown** - Properly waits for background tasks/goroutines to complete
- **Database Migrations** - Uses migrate tool for schema management
- **Vendored Dependencies** - All dependencies are vendored for reproducible builds
- **Custom Validator** - Input validation package for data integrity
- **Julienschmidt/httprouter** - High-performance HTTP router

## Prerequisites

- Go 1.24+ 
- PostgreSQL 12+
- Make (optional, for using Makefile commands)

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/Rif-7/movie-api
cd movie-api
```

### 2. Database setup

Create a PostgreSQL database and run migrations:

```bash
# Set up database connection string
export MOVIE_API_DB_DSN="postgres://user:password@localhost/movie_db"

# Run migrations
make db/migrations/up
```

### 3. Environment variables

Configure Mailtrap credentials:

```bash
export MAILTRAP_SMTP_HOST="smtp.mailtrap.io"
export MAILTRAP_SMTP_PORT="2525"
export MAILTRAP_SMTP_USERNAME="your_username"
export MAILTRAP_SMTP_PASSWORD="your_password"
export MAILTRAP_SMTP_SENDER="Movie API <no-reply@movieapi.com>"
```

### 4. Run the application

```bash
# Using Makefile
make run/api

# Or directly with go
go run ./cmd/api -db-dsn="postgres://user:password@localhost/movie_db"
```

## API Endpoints

### Authentication

All movie endpoints require authentication using Bearer tokens. Include the token in the Authorization header:

```
Authorization: Bearer <your-token>
```

### Base URL

```
http://localhost:4000
```

---

## Endpoints

### Health Check

**GET** `/v1/healthcheck`

Check API health status.

**Response:** `200 OK`

```json
{
  "status": "available",
  "environment": "development",
  "version": "1.0.0"
}
```

---

### Movies

#### Create Movie

**POST** `/v1/movies`

**Permission Required:** `movies:write`

**Request Body:**
```json
{
  "title": "The Shawshank Redemption",
  "year": 1994,
  "runtime": "142 mins",
  "genres": ["drama", "crime"]
}
```

**Response:** `201 Created`

```json
{
  "movie": {
    "id": 1,
    "title": "The Shawshank Redemption",
    "year": 1994,
    "runtime": "142 mins",
    "genres": ["drama", "crime"],
    "version": 1
  }
}
```

**Validation Rules:**
- `title`: Required, max 500 bytes
- `year`: Required, between 1888 and current year
- `runtime`: Required, positive integer (format: "XX mins")
- `genres`: Required, 1-5 genres, no duplicates

---

#### Get Movie

**GET** `/v1/movies/:id`

**Permission Required:** `movies:read`

**Response:** `200 OK`

```json
{
  "movie": {
    "id": 1,
    "title": "The Shawshank Redemption",
    "year": 1994,
    "runtime": "142 mins",
    "genres": ["drama", "crime"],
    "version": 1
  }
}
```

---

#### List Movies

**GET** `/v1/movies`

**Permission Required:** `movies:read`

**Query Parameters:**
- `page` (default: 1) - Page number
- `page_size` (default: 20, max: 100) - Results per page
- `sort` (default: id) - Sort by field (id, title, year, runtime). Prefix with `-` for descending
- `title` - Fulltext search on title
- `genres` - Filter by genres (comma-separated)

**Example:**
```
GET /v1/movies?page=1&page_size=10&sort=title&title=shawshank&genres=drama,crime
```

**Response:** `200 OK`

```json
{
  "movies": [
    {
      "id": 1,
      "title": "The Shawshank Redemption",
      "year": 1994,
      "runtime": "142 mins",
      "genres": ["drama", "crime"],
      "version": 1
    }
  ],
  "metadata": {
    "current_page": 1,
    "page_size": 20,
    "first_page": 1,
    "last_page": 5,
    "total_records": 100
  }
}
```

**Sort Options:**
- `id`, `-id`
- `title`, `-title`
- `year`, `-year`
- `runtime`, `-runtime`

---

#### Update Movie

**PATCH** `/v1/movies/:id`

**Permission Required:** `movies:write`

**Note:** Supports partial updates - only send fields that need to be updated.

**Request Body:**
```json
{
  "title": "The Shawshank Redemption",
  "year": 1994
}
```

**Response:** `200 OK`

```json
{
  "movie": {
    "id": 1,
    "title": "The Shawshank Redemption",
    "year": 1994,
    "runtime": "142 mins",
    "genres": ["drama", "crime"],
    "version": 2
}
```

**Error:** `409 Conflict` - Edit conflict (optimistic concurrency control)

---

#### Delete Movie

**DELETE** `/v1/movies/:id`

**Permission Required:** `movies:write`

**Response:** `200 OK`

```json
{
  "message": "movie successfully deleted"
}
```

---

### User Management

#### Register User

**POST** `/v1/users`

**Note:** User registration is public (no authentication required).

**Request Body:**
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "password123"
}
```

**Response:** `202 Accepted`

```json
{
  "user": {
    "id": 1,
    "created_at": "2023-01-01T00:00:00Z",
    "name": "John Doe",
    "email": "john@example.com",
    "activated": false
  }
}
```

**Note:** An activation email will be sent to the user with an activation token.

**Validation Rules:**
- `name`: Required, max 500 bytes
- `email`: Required, must be valid email format
- `password`: Required, 8-72 bytes

---

#### Activate User

**PUT** `/v1/users/activated`

**Request Body:**
```json
{
  "token": "activation-token-from-email"
}
```

**Response:** `200 OK`

```json
{
  "user": {
    "id": 1,
    "created_at": "2023-01-01T00:00:00Z",
    "name": "John Doe",
    "email": "john@example.com",
    "activated": true
  }
}
```

**Token Expiry:** 72 hours

---

#### Update Password

**PUT** `/v1/users/password`

**Request Body:**
```json
{
  "token": "password-reset-token-from-email",
  "password": "newpassword123"
}
```

**Response:** `200 OK`

```json
{
  "message": "your password was successfully reset"
}
```

**Token Expiry:** 45 minutes

---

### Authentication

#### Get Authentication Token

**POST** `/v1/tokens/authentication`

**Request Body:**
```json
{
  "email": "john@example.com",
  "password": "password123"
}
```

**Response:** `201 Created`

```json
{
  "authentication_token": {
    "token": "ABC123...XYZ",
    "expiry": "2024-01-02T00:00:00Z"
  }
}
```

**Token Expiry:** 24 hours

**Note:** Use this token in the Authorization header for authenticated requests.

**Error:** `401 Unauthorized` - Invalid credentials

---

#### Request Password Reset

**POST** `/v1/tokens/password-reset`

**Request Body:**
```json
{
  "email": "john@example.com"
}
```

**Response:** `202 Accepted`

```json
{
  "message": "an email will be sent to you containing password reset instructions"
}
```

**Note:** An email with a password reset token will be sent to the user.

**Token Expiry:** 45 minutes

---

### Metrics

**GET** `/debug/vars`

View application metrics in JSON format.

**Response:** `200 OK`

```json
{
  "cmdline": ["/path/to/binary"],
  "database": {
    "MaxOpenConnections": 25,
    "OpenConnections": 2,
    "InUse": 1,
    "Idle": 1
  },
  "goroutines": 42,
  "memstats": {...},
  "timestamp": 1234567890,
  "total_processing_time_μs": 1234567,
  "total_requests_received": 1000,
  "total_responses_sent": 1000,
  "total_responses_sent_by_status": {
    "200": 950,
    "201": 50
  },
  "version": "1.0.0"
}
```

---

## Command Line Flags

The API supports various configuration flags:

| Flag | Description | Default |
|------|-------------|---------|
| `-port` | API server port | 4000 |
| `-env` | Environment (development\|staging\|production) | development |
| `-db-dsn` | PostgreSQL connection string | (required) |
| `-db-max-open-conns` | Max open connections | 25 |
| `-db-max-idle-conns` | Max idle connections | 25 |
| `-db-max-idle-time` | Max idle time | 15m |
| `-limiter-rps` | Rate limiter requests per second | 2 |
| `-limiter-burst` | Rate limiter burst size | 4 |
| `-limiter-enabled` | Enable rate limiter | true |
| `-smtp-host` | SMTP host | (from env) |
| `-smtp-port` | SMTP port | 2525 |
| `-smtp-username` | SMTP username | (from env) |
| `-smtp-password` | SMTP password | (from env) |
| `-smtp-sender` | SMTP sender email | (from env) |
| `-cors-trusted-origins` | CORS trusted origins (space-separated) | "" |
| `-version` | Display version and exit | false |

**Example:**

```bash
go run ./cmd/api \
  -port=4000 \
  -env=production \
  -db-dsn="postgres://user:pass@localhost/db" \
  -limiter-rps=10 \
  -limiter-burst=20 \
  -cors-trusted-origins="https://example.com https://app.example.com"
```

---

## Rate Limiting

The API implements per-IP token bucket rate limiting.

**Default Limits:**
- Requests per second: 2
- Burst size: 4

**Response:** `429 Too Many Requests`

```json
{
  "error": "rate limit exceeded"
}
```

---

## Error Responses

All errors follow a consistent format:

```json
{
  "error": {
    "message": "specific error message"
  }
}
```

**For validation errors:**

```json
{
  "error": {
    "message": "validation failed",
    "errors": {
      "field_name": ["error message"]
    }
  }
}
```

### Common Status Codes

- `200 OK` - Success
- `201 Created` - Resource created
- `202 Accepted` - Request accepted (async processing)
- `400 Bad Request` - Invalid request
- `401 Unauthorized` - Authentication required/invalid credentials
- `403 Forbidden` - Permission denied
- `404 Not Found` - Resource not found
- `409 Conflict` - Edit conflict (optimistic concurrency)
- `422 Unprocessable Entity` - Validation failed
- `429 Too Many Requests` - Rate limit exceeded
- `500 Internal Server Error` - Server error

---

## Development

### Running Tests

```bash
# Run all tests
make audit

# Or manually
go test -race ./...
```

### Code Quality

```bash
# Format code
go fmt ./...

# Vet code
go vet ./...

# Static check
go tool staticcheck ./...

# Run all quality checks
make audit
```

### Database Management

```bash
# Connect to database
make db/psql

# Create new migration
make db/migrations/new name=create_permissions_table

# Apply migrations
make db/migrations/up
```

### Building

```bash
# Build for current platform
make build/api

# Or manually
go build -ldflags='-s' -o=./bin/api ./cmd/api
```

### Vendoring Dependencies

```bash
make tidy
```

---

## Project Structure

```
movie-api/
├── cmd/
│   └── api/              # Application entry point
│       ├── main.go       # Application initialization
│       ├── routes.go     # Route definitions
│       ├── server.go     # HTTP server setup
│       ├── middleware.go # Middleware (auth, rate limit, CORS, etc.)
│       ├── movies.go     # Movie handlers
│       ├── users.go      # User handlers
│       ├── tokens.go     # Token handlers
│       ├── healthcheck.go
│       ├── helpers.go
│       ├── errors.go
│       └── context.go
├── internal/
│   ├── data/             # Data layer (models, database)
│   │   ├── models.go
│   │   ├── movies.go
│   │   ├── users.go
│   │   ├── tokens.go
│   │   ├── permissions.go
│   │   ├── filters.go
│   │   └── runtime.go
│   ├── mailer/           # Email functionality
│   │   ├── mailer.go
│   │   └── templates/
│   └── validator/        # Input validation
│       └── validator.go
├── migrations/           # Database migrations
└── Makefile             # Development commands
```

---

## Security Features

- **Password Hashing** - Uses bcrypt with cost factor 12
- **Token-based Auth** - SHA-256 hashed tokens stored in database
- **CORS Protection** - Configurable trusted origins
- **Rate Limiting** - Per-IP rate limiting to prevent abuse
- **Input Validation** - Comprehensive input validation
- **Optimistic Concurrency** - Prevents race conditions
- **SQL Injection Protection** - Parameterized queries
- **Graceful Shutdown** - Proper cleanup on shutdown

---

## License

MIT

