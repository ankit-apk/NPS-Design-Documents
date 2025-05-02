# 🏛️ Architecture & Design Rulebook

## 1. 📦 Project Structure

```
/bookstore
│
├── api/                      # gRPC .proto definitions
├── cmd/                      # Application entrypoints (main.go)
│   └── server/
│
├── internal/                 # Business domains (modular)
│   └── auth/
│       ├── handler/          # gRPC (or HTTP) handlers
│       ├── service/          # Core business logic (UseCases)
│       ├── repo/             # Data access logic (DB, cache, etc.)
│       └── model.go          # Domain models (entities/value objects)
│
├── pkg/
│   ├── errors/               # Custom AppError + gRPC mapping
│   └── logger/               # Centralized logging
│
├── test/                     # Integration tests
└── go.mod
```

---

## 2. 🧱 Clean Architecture Principles

| Layer     | Description                            | Depends On  |
| --------- | -------------------------------------- | ----------- |
| `model`   | Domain entities                        | None        |
| `service` | Business logic (Use Cases)             | model, repo |
| `repo`    | Interface + Implementation for storage | model       |
| `handler` | gRPC / REST logic                      | service     |
| `api`     | Generated gRPC code                    | Proto       |

- **Service layer never depends on transport (gRPC, REST).**
- **Repositories are interfaces injected via constructor (DI).**
- **Handlers handle gRPC codes and translate errors.**

---

## 3. 🧾 Proto Rules

- Always keep `option go_package = "bookstore/api"` consistent.
- Use **snake_case** for field names in `.proto`.
- Define separate request/response messages even if empty.
- Add **comments** for all fields for future doc-gen.

---

## 4. ❌ Error Handling Rules

### Define all AppErrors in `pkg/errors/errors.go`:

```go
var ErrInvalidOTP = &AppError{
	Code:    "INVALID_OTP",
	Reason:  "OTP_MISMATCH",
	Message: "OTP does not match",
	GRPC:    codes.InvalidArgument,
}
```

### Always return `AppError` from service layer.

Handlers convert `AppError` → gRPC `status.Status` with:

```go
if appErr, ok := err.(interface{ GRPCStatus() *status.Status }); ok {
	return nil, appErr.GRPCStatus().Err()
}
```

---

## 5. 🧪 Testing Rules

- Use **table-driven tests** for unit tests.
- Mock the `Repository` in service tests.
- Test `handler` layer using gRPC test server or integration tests.

---

## 6. 📊 Logging

- Use `pkg/logger` with structured logging (`logrus` / `zap`).
- Do **not log** inside `model` or `repo` unless fatal.
- Log request/response payloads only in handlers (with masking for PII).

---

## 7. 🔗 Dependency Injection

- All dependencies should be injected via constructor.
- No use of global variables or `init()` magic.
- Compose dependencies only in `cmd/server/main.go`.

---

## 8. 🔄 Modularity Guidelines

- One **domain per folder** under `internal/`.

- If a module grows large, split further:

  ```
  internal/auth/
    ├── service/
    ├── repository/
    ├── usecase/
    └── delivery/
  ```

- Never cross-call domains directly. Use interfaces.

- For cross-module calls, define usecase interfaces and call via service injection.

---

## 9. 🚀 Server Bootstrapping

All services should follow this pattern:

```go
repo := auth.NewInMemoryRepo()
svc := auth.NewService(repo)
grpcServer := grpc.NewServer()
api.RegisterAuthServiceServer(grpcServer, handler.New(svc))
```

Main responsibilities:

- Compose dependencies
- Listen on ports
- Register gRPC/HTTP endpoints

---

## 10. 🛡️ Security Rules

- Never return raw errors to clients.
- Validate all input in `handler` layer (phone, email formats, etc).
- Avoid business logic in handlers.
- If handling sensitive info (like OTP), mask in logs.

---

## 11. 📚 Naming Conventions

- Interface: `Service`, `Repository`
- Implementations: `AuthService`, `InMemoryRepo`
- gRPC methods: `VerifyOTP`, `SendEmail`
- Messages: `VerifyOTPRequest`, `VerifyOTPResponse`
- Errors: `ErrInvalidOTP`, `ErrUserNotFound`

---

## 12. 📂 Code Reuse

- Shared utilities like `errors`, `logger`, `auth/token` go in `pkg/`
- Domain-specific logic stays inside that module (`internal/auth`)
- Avoid circular imports: domain A → B → A

---

## 13. 🚧 Extending the System

If you want to add a new domain module, follow these steps:

1. Create a folder: `internal/user/`
2. Add `model.go`, `service.go`, `repository.go`, `handler/`
3. Define interface in `repository.go`
4. Implement `service.go` to use repo
5. Add `.proto` definitions
6. Register it in `cmd/server/main.go`

---

## 14. ✅ API Contract Rules

- Always return either full success OR structured error (never mix).
- Prefer one message per response for evolution.
- Don’t return bools like `ok`; instead return valid data or error.
