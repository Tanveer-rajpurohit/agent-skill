---
name: go-backend
description: >
  Write idiomatic, production Go backend services matching Tanveer's actual stack.
  Trigger for: building HTTP services with chi, sqlc query patterns, pgx/pgxpool
  database access, JWT auth (golang-jwt/jwt/v5, access + refresh tokens), bcrypt
  passwords, rate limiting (golang.org/x/time/rate), S3 uploads (aws-sdk-go-v2),
  Redis (go-redis/v9), background job queues (asynq), WebSockets (gorilla/websocket),
  gRPC (grpc-go + protobuf), CLI tools (cobra), context cancellation, concurrency
  (goroutines, channels, errgroup, sync), graceful shutdown, error handling,
  golang-migrate migrations, project structure, air hot reload setup, and "is this
  idiomatic Go" reviews. Trigger on: "write a Go handler", "chi route", "sqlc
  query", "JWT middleware", "Go project structure", "rate limit", "asynq worker",
  "WebSocket hub", "gRPC service".
---

# Go Backend Skill — Tanveer's Exact Stack

You write Go the way Tanveer already works, plus the best practices that level it up.
Stack: **chi + pgx/v5 + sqlc + golang-migrate + golang-jwt + bcrypt +
golang.org/x/time/rate + aws-sdk-go-v2 + go-redis/v9 + asynq + gorilla/websocket +
grpc-go + cobra + air**.

---

## Project structure (match this exactly)

```
<project-name>/
├── cmd/server/main.go              # wiring, config, graceful shutdown — nothing else
├── db/
│   ├── migrations/                 # *.up.sql / *.down.sql  (golang-migrate)
│   └── query/                      # *.sql with sqlc annotations
├── internal/
│   ├── auth/
│   │   ├── jwt.go                  # GenerateAccessToken, GenerateRefreshToken, ValidateToken
│   │   └── middleware.go           # RequireAuth, RequireAdmin chi middlewares
│   ├── config/
│   │   ├── db.go                   # ConnectDB() *pgxpool.Pool
│   │   └── redis.go                # ConnectRedis() *redis.Client
│   ├── db/                         # ← sqlc generated, NEVER edit manually
│   ├── handlers/                   # handler structs per domain
│   ├── middleware/                 # shared middleware (ratelimit, etc.)
│   ├── routes/routes.go            # SetupRouter wires handlers
│   ├── storage/s3.go               # S3Client
│   └── utils/json.go               # ResponseWithJSON, ResponseWithError
├── .env  /  .env.example
├── .air.toml
├── sqlc.yaml
├── go.mod  /  go.sum
```

---

## Bootstrap (do this once per project)

```bash
go mod init github.com/Tanveer-rajpurohit/<project-name>

# Core
go get github.com/go-chi/chi/v5 github.com/go-chi/cors github.com/joho/godotenv

# DB
go get github.com/jackc/pgx/v5
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Auth
go get github.com/golang-jwt/jwt/v5 golang.org/x/crypto

# Dev
go install github.com/air-verse/air@latest && air init
```

---

## `cmd/server/main.go` — the right way

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/Tanveer-rajpurohit/<project>/internal/config"
    "github.com/Tanveer-rajpurohit/<project>/internal/db"
    "github.com/Tanveer-rajpurohit/<project>/internal/routes"
    "github.com/go-chi/chi/v5"
    "github.com/go-chi/cors"
    "github.com/joho/godotenv"
)

func main() {
    if err := godotenv.Load(); err != nil {
        log.Println("no .env file, using system env")
    }

    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    pool := config.ConnectDB()
    defer pool.Close()

    router := chi.NewRouter()
    router.Use(cors.Handler(cors.Options{
        AllowedOrigins:   []string{os.Getenv("ALLOWED_ORIGIN")},  // not "*" in prod
        AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"},
        AllowedHeaders:   []string{"Accept", "Authorization", "Content-Type"},
        AllowCredentials: true,
        MaxAge:           300,
    }))

    v1 := chi.NewRouter()
    routes.SetupRouter(v1, db.New(pool))
    router.Mount("/api/v1", v1)

    srv := &http.Server{
        Handler:      router,
        Addr:         ":" + port,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,   // longer for file uploads / streaming
        IdleTimeout:  120 * time.Second,
    }

    go func() {
        log.Printf("server listening on :%s", port)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // Graceful shutdown on Ctrl-C / SIGTERM
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("shutdown error: %v", err)
    }
    log.Println("server stopped")
}
```

Key improvements vs the original: server timeouts set (no DoS exposure), CORS
`AllowedOrigins` uses env (not hardcoded `*`), graceful shutdown so in-flight
requests finish cleanly.

---

## sqlc workflow

**`sqlc.yaml`:**
```yaml
version: "2"
sql:
  - schema: "db/migrations"
    queries: "db/query"
    engine: "postgresql"
    gen:
      go:
        package: "db"
        out: "internal/db"
        sql_package: "pgx/v5"
        emit_json_tags: true
        json_tags_case_style: "snake"
```

**`db/query/users.sql`:**
```sql
-- name: GetUserByID :one
SELECT id, name, email, role, created_at FROM users WHERE id = $1;

-- name: GetUserByEmail :one
SELECT id, name, email, role, password FROM users WHERE email = $1;

-- name: CreateUser :one
INSERT INTO users (name, email, role, password)
VALUES ($1, $2, $3, $4)
RETURNING id, name, email, role, created_at;

-- name: UpdateUser :one
UPDATE users SET name = $2, email = $3 WHERE id = $1
RETURNING id, name, email, role, created_at;

-- name: DeleteUser :exec
DELETE FROM users WHERE id = $1;
```

```bash
sqlc generate          # re-run every time you change a query or migration
```

Generated code in `internal/db/` gives you: `Queries` struct, typed params, typed
row structs, fully parameterized SQL — no string interpolation, no scan mistakes.

---

## Handler struct pattern (how you use it)

```go
// internal/handlers/user.go
package handlers

import (
    "errors"
    "net/http"

    "github.com/Tanveer-rajpurohit/<project>/internal/db"
    "github.com/Tanveer-rajpurohit/<project>/internal/utils"
    "github.com/go-chi/chi/v5"
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgtype"
)

type UserHandler struct {
    Q *db.Queries
}

func (h *UserHandler) GetByID(w http.ResponseWriter, r *http.Request) {
    idStr := chi.URLParam(r, "id")
    var id pgtype.UUID
    if err := id.Scan(idStr); err != nil {          // ← ALWAYS handle parse errors
        utils.ResponseWithError(w, http.StatusBadRequest, "invalid id")
        return
    }

    user, err := h.Q.GetUserByID(r.Context(), id)  // ← r.Context(), not context.Background()
    switch {
    case errors.Is(err, pgx.ErrNoRows):
        utils.ResponseWithError(w, http.StatusNotFound, "user not found")
    case err != nil:
        utils.ResponseWithError(w, http.StatusInternalServerError, "internal server error")
    default:
        utils.ResponseWithJSON(w, http.StatusOK, user)
    }
}

func (h *UserHandler) Delete(w http.ResponseWriter, r *http.Request) {
    idStr := chi.URLParam(r, "id")
    var id pgtype.UUID
    if err := id.Scan(idStr); err != nil {
        utils.ResponseWithError(w, http.StatusBadRequest, "invalid id")
        return
    }

    if err := h.Q.DeleteUser(r.Context(), id); err != nil {  // ← NEVER ignore delete error
        utils.ResponseWithError(w, http.StatusInternalServerError, "delete failed")
        return
    }
    utils.ResponseWithJSON(w, http.StatusNoContent, nil)
}
```

**Two bugs to fix in every handler vs the p1 originals:**
1. `context.Background()` → `r.Context()` — so the request's deadline/cancellation
   propagates to the DB query. If the client disconnects, the query cancels.
2. Ignored errors (`id, _ := strconv.Atoi(...)`, `h.Q.DeleteUser(...)` with no err
   check) → always handle. An ignored error is a silent corruption or wrong response.

---

## Auth — JWT access + refresh (your exact pattern, improved)

```go
// internal/auth/jwt.go
package auth

import (
    "errors"
    "fmt"
    "os"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
    UserID string `json:"user_id"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

const accessTTL  = 15 * time.Minute
const refreshTTL = 7 * 24 * time.Hour

func GenerateAccessToken(userID, role string) (string, error) {
    return sign(userID, role, accessTTL, "JWT_SECRET")
}

func GenerateRefreshToken(userID, role string) (string, error) {
    return sign(userID, role, refreshTTL, "JWT_REFRESH_SECRET")
}

func sign(userID, role string, ttl time.Duration, envKey string) (string, error) {
    secret := os.Getenv(envKey)
    if secret == "" {
        return "", fmt.Errorf("env %s not set", envKey)
    }
    claims := Claims{
        UserID: userID,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(ttl)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    return jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString([]byte(secret))
}

func ValidateToken(tokenStr string, isRefresh bool) (*Claims, error) {
    envKey := "JWT_SECRET"
    if isRefresh {
        envKey = "JWT_REFRESH_SECRET"
    }
    secret := os.Getenv(envKey)
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return []byte(secret), nil
    })
    if err != nil || !token.Valid {
        return nil, errors.New("invalid token")
    }
    return token.Claims.(*Claims), nil
}
```

```go
// internal/auth/middleware.go
package auth

import (
    "context"
    "net/http"
    "strings"

    "github.com/Tanveer-rajpurohit/<project>/internal/utils"
)

type contextKey string
const ClaimsKey contextKey = "claims"

func RequireAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        header := r.Header.Get("Authorization")
        if !strings.HasPrefix(header, "Bearer ") {
            utils.ResponseWithError(w, http.StatusUnauthorized, "unauthorized")
            return
        }
        claims, err := ValidateToken(strings.TrimPrefix(header, "Bearer "), false)
        if err != nil {
            utils.ResponseWithError(w, http.StatusUnauthorized, "invalid token")
            return
        }
        next.ServeHTTP(w, r.WithContext(context.WithValue(r.Context(), ClaimsKey, claims)))
    })
}

func RequireAdmin(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        claims, ok := r.Context().Value(ClaimsKey).(*Claims)
        if !ok || claims.Role != "admin" {
            utils.ResponseWithError(w, http.StatusForbidden, "forbidden")
            return
        }
        next.ServeHTTP(w, r)
    })
}

// Helper to pull claims in handlers
func GetClaims(r *http.Request) (*Claims, bool) {
    claims, ok := r.Context().Value(ClaimsKey).(*Claims)
    return claims, ok
}
```

---

## Rate limiting (your in-memory store pattern)

```go
// internal/middleware/ratelimit.go  — matches p2, with cleanup goroutine lifecycle fix
package middleware

import (
    "net"
    "net/http"
    "sync"
    "time"

    "github.com/Tanveer-rajpurohit/<project>/internal/auth"
    "github.com/Tanveer-rajpurohit/<project>/internal/utils"
    "golang.org/x/time/rate"
)

type Store struct {
    mu       sync.Mutex
    limiters map[string]*entry
    interval time.Duration
    burst    int
    stop     chan struct{}
}

type entry struct {
    limiter  *rate.Limiter
    lastSeen time.Time
}

func NewStore(refill time.Duration, burst int) *Store {
    s := &Store{
        limiters: make(map[string]*entry),
        interval: refill,
        burst:    burst,
        stop:     make(chan struct{}),
    }
    go s.cleanupLoop()
    return s
}

func (s *Store) Stop() { close(s.stop) }   // call in graceful shutdown

func (s *Store) cleanupLoop() {
    ticker := time.NewTicker(2 * time.Minute)
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            s.mu.Lock()
            for id, e := range s.limiters {
                if time.Since(e.lastSeen) > 5*time.Minute {
                    delete(s.limiters, id)
                }
            }
            s.mu.Unlock()
        case <-s.stop:
            return
        }
    }
}

func (s *Store) get(key string) *rate.Limiter {
    s.mu.Lock()
    defer s.mu.Unlock()
    e, ok := s.limiters[key]
    if !ok {
        e = &entry{limiter: rate.NewLimiter(rate.Every(s.interval), s.burst)}
        s.limiters[key] = e
    }
    e.lastSeen = time.Now()
    return e.limiter
}

func (s *Store) Limit(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        key := ""
        if claims, ok := auth.GetClaims(r); ok {
            key = claims.UserID
        }
        if key == "" {
            ip, _, err := net.SplitHostPort(r.RemoteAddr)
            if err != nil {
                ip = r.RemoteAddr
            }
            key = ip
        }
        if !s.get(key).Allow() {
            w.Header().Set("Retry-After", "60")
            utils.ResponseWithError(w, http.StatusTooManyRequests, "too many requests")
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

For distributed rate limiting (multiple instances) — use Redis instead of the
in-memory map. See `system-design` PATTERNS.md for the sliding window Redis impl.

---

## S3 (aws-sdk-go-v2 — streaming, no RAM load)

```go
// internal/storage/s3.go
package storage

import (
    "context"
    "fmt"
    "io"

    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

type S3Client struct {
    client *s3.Client
    bucket string
    region string
}

func NewS3Client(bucket, region string) (*S3Client, error) {
    cfg, err := config.LoadDefaultConfig(context.Background(), config.WithRegion(region))
    // In prod: use IAM role (no static creds). Locally: env AWS_ACCESS_KEY_ID/SECRET.
    if err != nil {
        return nil, fmt.Errorf("load aws config: %w", err)
    }
    return &S3Client{client: s3.NewFromConfig(cfg), bucket: bucket, region: region}, nil
}

// Upload streams from io.Reader — never loads the full file into RAM.
// Pass r.Body directly for multipart uploads.
func (s *S3Client) Upload(ctx context.Context, key, contentType string, body io.Reader) (string, error) {
    _, err := s.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket:      aws.String(s.bucket),
        Key:         aws.String(key),
        Body:        body,
        ContentType: aws.String(contentType),
    })
    if err != nil {
        return "", fmt.Errorf("s3 upload %s: %w", key, err)
    }
    return fmt.Sprintf("https://%s.s3.%s.amazonaws.com/%s", s.bucket, s.region, key), nil
}

func (s *S3Client) Download(ctx context.Context, key string) (io.ReadCloser, error) {
    out, err := s.client.GetObject(ctx, &s3.GetObjectInput{
        Bucket: aws.String(s.bucket),
        Key:    aws.String(key),
    })
    if err != nil {
        return nil, fmt.Errorf("s3 download %s: %w", key, err)
    }
    return out.Body, nil   // caller MUST close this
}
```

For large uploads use `s3manager.Uploader` (multipart, parallelized, auto-retry):
```go
import "github.com/aws/aws-sdk-go-v2/feature/s3/manager"
uploader := manager.NewUploader(s.client)
_, err = uploader.Upload(ctx, &s3.PutObjectInput{Bucket: ..., Key: ..., Body: reader})
```

---

## Asynq — background job queue (p2 pattern)

```go
// install: go get github.com/hibiken/asynq

// internal/jobs/tasks.go
package jobs

const TaskProcessImage = "image:process"

type ProcessImagePayload struct {
    AssetID     string `json:"asset_id"`
    S3Key       string `json:"s3_key"`
    Mode        string `json:"mode"`        // "compress" | "original" | "resize"
    TargetFormat string `json:"target_format"` // "webp" | "jpeg"
}

// Enqueue from handler
func EnqueueProcessImage(client *asynq.Client, p ProcessImagePayload) error {
    payload, err := json.Marshal(p)
    if err != nil {
        return fmt.Errorf("marshal payload: %w", err)
    }
    task := asynq.NewTask(TaskProcessImage, payload,
        asynq.MaxRetry(3),
        asynq.Timeout(5*time.Minute),
    )
    _, err = client.Enqueue(task, asynq.Queue("media"))
    return err
}

// Worker (run in a separate goroutine or binary)
func NewWorkerServer(redisOpt asynq.RedisClientOpt, queries *db.Queries, s3 *storage.S3Client) *asynq.Server {
    srv := asynq.NewServer(redisOpt, asynq.Config{
        Concurrency: 10,
        Queues:      map[string]int{"media": 6, "default": 3, "low": 1},
    })
    mux := asynq.NewServeMux()
    mux.HandleFunc(TaskProcessImage, handleProcessImage(queries, s3))
    go func() {
        if err := srv.Run(mux); err != nil {
            log.Fatalf("asynq server error: %v", err)
        }
    }()
    return srv
}

func handleProcessImage(q *db.Queries, s3 *storage.S3Client) asynq.HandlerFunc {
    return func(ctx context.Context, t *asynq.Task) error {
        var p ProcessImagePayload
        if err := json.Unmarshal(t.Payload(), &p); err != nil {
            return fmt.Errorf("unmarshal: %w", err)
        }
        // Download → process → upload → update DB
        body, err := s3.Download(ctx, p.S3Key)
        if err != nil { return err }
        defer body.Close()
        // ... image processing ...
        return nil
    }
}
```

---

## WebSocket hub pattern (p3 — gorilla/websocket)

```go
// install: go get github.com/gorilla/websocket

type Hub struct {
    rooms      map[string]map[*Client]bool
    broadcast  chan Message
    register   chan *Client
    unregister chan *Client
    mu         sync.RWMutex
}

type Client struct {
    hub    *Hub
    roomID string
    conn   *websocket.Conn
    send   chan []byte
}

// Run in a goroutine — single goroutine owns all map mutations (no locking needed)
func (h *Hub) Run(ctx context.Context) {
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            if h.rooms[client.roomID] == nil {
                h.rooms[client.roomID] = make(map[*Client]bool)
            }
            h.rooms[client.roomID][client] = true
            h.mu.Unlock()

        case client := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.rooms[client.roomID][client]; ok {
                delete(h.rooms[client.roomID], client)
                close(client.send)
            }
            h.mu.Unlock()

        case msg := <-h.broadcast:
            h.mu.RLock()
            for client := range h.rooms[msg.RoomID] {
                select {
                case client.send <- msg.Data:
                default:
                    close(client.send)
                    delete(h.rooms[msg.RoomID], client)
                }
            }
            h.mu.RUnlock()

        case <-ctx.Done():
            return   // goroutine exits cleanly on shutdown
        }
    }
}

// Each client runs two goroutines: reader + writer
func (c *Client) ReadPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()
    c.conn.SetReadLimit(4096)
    c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })
    for {
        _, msg, err := c.conn.ReadMessage()
        if err != nil { return }
        c.hub.broadcast <- Message{RoomID: c.roomID, Data: msg}
    }
}
```

For horizontal scaling (p4): replace in-process hub channels with Redis Pub/Sub —
each instance subscribes to the room topic, publishes to Redis when it receives a
message, and Redis fans out to all instances.

---

## gRPC service (p4 pattern)

```bash
# Install
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
go get google.golang.org/grpc
```

```protobuf
// proto/auth/v1/auth.proto
syntax = "proto3";
option go_package = "github.com/Tanveer-rajpurohit/<project>/proto/auth/v1;authv1";

service AuthService {
  rpc ValidateToken (ValidateTokenRequest) returns (ValidateTokenResponse);
}
message ValidateTokenRequest { string token = 1; }
message ValidateTokenResponse { bool valid = 1; string user_id = 2; string role = 3; }
```

```bash
protoc --go_out=. --go-grpc_out=. proto/auth/v1/auth.proto
```

```go
// Server implementation
type authServer struct {
    authv1.UnimplementedAuthServiceServer
}

func (s *authServer) ValidateToken(ctx context.Context, req *authv1.ValidateTokenRequest) (*authv1.ValidateTokenResponse, error) {
    claims, err := auth.ValidateToken(req.Token, false)
    if err != nil {
        return &authv1.ValidateTokenResponse{Valid: false}, nil
    }
    return &authv1.ValidateTokenResponse{Valid: true, UserId: claims.UserID, Role: claims.Role}, nil
}
```

---

## CLI with cobra (p5 pattern)

```go
// install: go get github.com/spf13/cobra

var rootCmd = &cobra.Command{Use: "admin"}

func init() {
    rootCmd.AddCommand(purgeCmd, metricsCmd)
}

var purgeCmd = &cobra.Command{
    Use:   "purge [queue]",
    Short: "Purge all jobs from an asynq queue",
    Args:  cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        inspector := asynq.NewInspector(asynq.RedisClientOpt{Addr: os.Getenv("REDIS_URL")})
        return inspector.DeleteAllPendingTasks(args[0])
    },
}
```

---

## Error handling (the rules)

```go
// Wrap with context as errors rise — %w preserves chain for errors.Is/As
return nil, fmt.Errorf("get user %s: %w", id, err)

// Sentinel for conditions callers branch on
var ErrNotFound = errors.New("not found")

// In handler — switch on sentinel, not string match
switch {
case errors.Is(err, pgx.ErrNoRows):   // or your ErrNotFound
    utils.ResponseWithError(w, 404, "not found")
case err != nil:
    utils.ResponseWithError(w, 500, "internal server error")
    // also: log.Printf("getUser: %v", err) — log here, don't return err detail to client
default:
    utils.ResponseWithJSON(w, 200, result)
}
```

---

## The bugs to kill in every handler

```go
// ❌ p1 bug 1: context.Background() — doesn't propagate request cancellation
user, err := h.Q.GetUser(context.Background(), id)
// ✅
user, err := h.Q.GetUser(r.Context(), id)

// ❌ p1 bug 2: ignored parse error
id, _ := strconv.Atoi(idStr)
// ✅ (or better — use UUID, not int IDs)
id, err := strconv.Atoi(idStr)
if err != nil { utils.ResponseWithError(w, 400, "invalid id"); return }

// ❌ p1 bug 3: delete error silently ignored
h.Q.DeleteUser(context.Background(), int32(id))
// ✅
if err := h.Q.DeleteUser(r.Context(), id); err != nil {
    utils.ResponseWithError(w, 500, "delete failed"); return
}

// ❌ Not found on GetUserByID returns 500
// ✅ errors.Is(err, pgx.ErrNoRows) → 404
```

---

## Migrations (golang-migrate)

```bash
# Create
migrate create -ext sql -dir db/migrations <name>

# Run
migrate -path db/migrations -database "$DATABASE_URL" up
migrate -path db/migrations -database "$DATABASE_URL" down 1

# Use UUIDs, not SERIAL — UUIDs are safe to expose in URLs, don't leak row count
CREATE TABLE users (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name       TEXT NOT NULL,
    email      TEXT UNIQUE NOT NULL,
    password   TEXT NOT NULL,
    role       TEXT NOT NULL DEFAULT 'user',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Common commands cheatsheet

| Task | Command |
|---|---|
| Dev server (hot reload) | `air` |
| Create migration | `migrate create -ext sql -dir db/migrations <name>` |
| Run migrations | `migrate -path db/migrations -database "$DATABASE_URL" up` |
| Rollback 1 migration | `migrate -path db/migrations -database "$DATABASE_URL" down 1` |
| Regenerate DB code | `sqlc generate` |
| Race detector | `go test -race ./...` |
| Lint | `golangci-lint run` |
| Build binary | `go build -o app ./cmd/server` |

---

## Anti-patterns

```
❌ context.Background() in handlers          → r.Context()
❌ Ignoring parse / delete errors            → always check err
❌ CORS AllowedOrigins: []string{"*"}        → use env var in prod
❌ No server timeouts                        → ReadTimeout/WriteTimeout/IdleTimeout
❌ SERIAL/int IDs exposed in URLs            → UUID (gen_random_uuid())
❌ Static AWS creds in code                  → IAM role in prod, env locally
❌ Loading full file into RAM for S3         → stream via io.Reader
❌ No graceful shutdown                      → signal.NotifyContext + srv.Shutdown
❌ Logging AND returning the same error      → log once at boundary, return generic msg
❌ Unbounded cleanup goroutine               → select on stop channel
```
