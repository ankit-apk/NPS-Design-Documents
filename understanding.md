## 📦 Project Overview: `bookstore/` (gRPC + Clean Architecture)

### 🗂️ Layered File & Call Hierarchy

```plaintext
bookstore/
└── cmd/
    └── server/
        └── main.go                    # App entrypoint
            ├── initializes → pkg/db/db.go
            ├── injects → internal/book/repository/repository.go
            │                  └── depends on → internal/book/model.go
            ├── injects → internal/book/service/service.go
            │                  ├── uses → repository interface
            │                  ├── uses → internal/book/model.go
            │                  └── uses → pkg/errors/errors.go
            ├── injects → internal/book/handler/grpc.go
            │                  ├── implements → api/book_grpc.pb.go
            │                  ├── calls → service layer
            │                  └── uses → pkg/errors/errors.go
            └── registers → api.BookServiceServer (gRPC)
```

---

## 📊 Layered Dependency Breakdown

| 🧱 Layer           | 🧾 Files                                          | 🔗 Depends On                                       |
| ------------------ | ------------------------------------------------- | --------------------------------------------------- |
| **Entrypoint**     | `cmd/server/main.go`                              | `db`, `repository`, `service`, `handler`, `api`     |
| **Transport**      | `internal/book/handler/grpc.go`                   | `service`, `api`, `errors`                          |
| **Application**    | `internal/book/service/service.go`                | `repository`, `model`, `errors`                     |
| **Persistence**    | `internal/book/repository/repository.go`          | `model`, `gorm`                                     |
| **Domain**         | `internal/book/model.go`                          | (pure domain, no dependencies)                      |
| **Infrastructure** | `pkg/db/db.go`                                    | `gorm`                                              |
| **Shared**         | `pkg/errors/errors.go`                            | `grpc/status`, `grpc/codes`, generic `errors` pkg   |
| **Protobuf/API**   | `api/book.proto`, `book.pb.go`, `book_grpc.pb.go` | Used by transport (handler), compiled from `.proto` |

---

## 🔄 Flow Overview (Call Graph)

```plaintext
main.go
│
├── Connects to Postgres via pkg/db/db.go
├── Initializes repository → repository uses model.go
├── Initializes service → depends on repository + model + errors
├── Initializes gRPC handler → calls service, handles gRPC API
├── Registers gRPC handler → api.RegisterBookServiceServer
└── Starts grpc.Server on :50051
```

---

## 📌 Design Principles Reinforced

- ✅ **Downward-only dependencies** (higher layers depend on lower, never the reverse)
- ✅ **Handler imports compiled `.proto` files** — other layers remain agnostic
- ✅ **Repository is interface-driven** — pluggable & mockable
- ✅ **Domain is isolated** — no external package imports
- ✅ **Shared concerns (e.g. error handling)** live in `pkg/`
