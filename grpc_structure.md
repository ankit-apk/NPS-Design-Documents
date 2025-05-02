## ✅ Step-by-Step: gRPC CRUD App with Clean Architecture (Modular Monolith in Mono-repo)

---

### 📁 Folder Structure

```
your-mono-repo/
├── protos/                        # ✅ Shared across all services
│   └── book/
│       └── book.proto             # gRPC service definitions
│
├── bookstore/                     # ✅ Modular monolith service
│   ├── cmd/
│   │   └── server/
│   │       └── main.go            # bootstrap file
│   ├── internal/
│   │   └── book/
│   │       ├── handler/           # gRPC handler
│   │       ├── service/           # business logic
│   │       ├── repository/        # db layer
│   │       └── model.go           # domain model
│   ├── pkg/
│   │   ├── db/                    # gorm setup
│   │   └── errors/                # global custom errors
│   └── go.mod
│
└── go.work                        # ✅ Optional workspace (recommended)
```

---

### 1️⃣ `protos/book/book.proto`

```proto
syntax = "proto3";

package book;

option go_package = "github.com/your-org/protos/book;bookpb";

service BookService {
  rpc CreateBook (CreateBookRequest) returns (BookResponse);
  rpc GetBook (GetBookRequest) returns (BookResponse);
  rpc ListBooks (Empty) returns (BookListResponse);
  rpc UpdateBook (UpdateBookRequest) returns (BookResponse);
  rpc DeleteBook (DeleteBookRequest) returns (Empty);
}

message Empty {}

message Book {
  uint32 id = 1;
  string title = 2;
  string author = 3;
}

message CreateBookRequest {
  string title = 1;
  string author = 2;
}

message UpdateBookRequest {
  uint32 id = 1;
  string title = 2;
  string author = 3;
}

message GetBookRequest {
  uint32 id = 1;
}

message DeleteBookRequest {
  uint32 id = 1;
}

message BookResponse {
  Book book = 1;
}

message BookListResponse {
  repeated Book books = 1;
}
```

### ✅ Generate Code

From the **root**:

```bash
protoc --go_out=. --go-grpc_out=. protos/book/book.proto
```

This generates `book.pb.go` and `book_grpc.pb.go` under `protos/book/` which will be imported in the service.

---

### 2️⃣ `internal/book/model.go`

```go
package book

type Book struct {
	ID     uint
	Title  string
	Author string
}
```

---

### 3️⃣ `internal/book/repository/repository.go`

```go
package repository

import (
	"bookstore/internal/book"
	"gorm.io/gorm"
)

type Repository interface {
	Create(book *book.Book) error
	Get(id uint) (*book.Book, error)
	List() ([]book.Book, error)
	Update(book *book.Book) error
	Delete(id uint) error
}

type repo struct {
	db *gorm.DB
}

func New(db *gorm.DB) Repository {
	return &repo{db}
}

func (r *repo) Create(b *book.Book) error {
	return r.db.Create(b).Error
}

func (r *repo) Get(id uint) (*book.Book, error) {
	var b book.Book
	err := r.db.First(&b, id).Error
	return &b, err
}

func (r *repo) List() ([]book.Book, error) {
	var books []book.Book
	err := r.db.Find(&books).Error
	return books, err
}

func (r *repo) Update(b *book.Book) error {
	return r.db.Save(b).Error
}

func (r *repo) Delete(id uint) error {
	return r.db.Delete(&book.Book{}, id).Error
}
```

---

### 4️⃣ `internal/book/service/service.go`

```go
package service

import (
	"bookstore/internal/book"
	"bookstore/pkg/errors"
)

type Service interface {
	Create(*book.Book) (*book.Book, error)
	Get(id uint) (*book.Book, error)
	List() ([]book.Book, error)
	Update(*book.Book) (*book.Book, error)
	Delete(id uint) error
}

type service struct {
	repo book.Repository
}

func New(repo book.Repository) Service {
	return &service{repo}
}

func (s *service) Create(b *book.Book) (*book.Book, error) {
	return b, s.repo.Create(b)
}

func (s *service) Get(id uint) (*book.Book, error) {
	b, err := s.repo.Get(id)
	if err != nil {
		return nil, errors.ErrBookNotFound
	}
	return b, nil
}

func (s *service) List() ([]book.Book, error) {
	return s.repo.List()
}

func (s *service) Update(b *book.Book) (*book.Book, error) {
	err := s.repo.Update(b)
	return b, err
}

func (s *service) Delete(id uint) error {
	return s.repo.Delete(id)
}
```

---

### 5️⃣ `internal/book/handler/grpc.go`

```go
package handler

import (
	"bookstore/internal/book"
	"bookstore/pkg/errors"
	bookpb "github.com/your-org/protos/book"
	"context"
)

type BookHandler struct {
	bookpb.UnimplementedBookServiceServer
	svc book.Service
}

func New(svc book.Service) *BookHandler {
	return &BookHandler{svc: svc}
}

func (h *BookHandler) GetBook(ctx context.Context, req *bookpb.GetBookRequest) (*bookpb.BookResponse, error) {
	b, err := h.svc.Get(uint(req.Id))
	if err != nil {
		return nil, errors.ToGRPC(err)
	}
	return &bookpb.BookResponse{
		Book: &bookpb.Book{
			Id:     b.ID,
			Title:  b.Title,
			Author: b.Author,
		},
	}, nil
}

// Implement CreateBook, ListBooks, UpdateBook, DeleteBook similarly
```

---

### 6️⃣ `pkg/errors/errors.go`

```go
package errors

import (
	"errors"

	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

var (
	ErrBookNotFound = errors.New("book not found")
	ErrInvalidInput = errors.New("invalid input")
)

func ToGRPC(err error) error {
	switch err {
	case ErrBookNotFound:
		return status.Error(codes.NotFound, err.Error())
	case ErrInvalidInput:
		return status.Error(codes.InvalidArgument, err.Error())
	default:
		return status.Error(codes.Internal, "internal error")
	}
}
```

---

### 7️⃣ `pkg/db/postgres.go`

```go
package db

import (
	"gorm.io/driver/postgres"
	"gorm.io/gorm"
)

func InitPostgres() *gorm.DB {
	dsn := "host=localhost user=postgres password=postgres dbname=bookstore port=5432 sslmode=disable"
	db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
	if err != nil {
		panic(err)
	}
	return db
}
```

---

### 8️⃣ `cmd/server/main.go`

```go
package main

import (
	"bookstore/internal/book/handler"
	"bookstore/internal/book/repository"
	"bookstore/internal/book/service"
	"bookstore/pkg/db"
	bookpb "github.com/your-org/protos/book"

	"google.golang.org/grpc"
	"log"
	"net"
)

func main() {
	dbConn := db.InitPostgres()
	bookRepo := repository.New(dbConn)
	bookSvc := service.New(bookRepo)
	bookHandler := handler.New(bookSvc)

	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("❌ failed to listen: %v", err)
	}

	grpcServer := grpc.NewServer()
	bookpb.RegisterBookServiceServer(grpcServer, bookHandler)

	log.Println("🚀 gRPC server running at :50051")
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("❌ failed to serve: %v", err)
	}
}
```

---

## ✅ Workspace Setup (`go.work`)

```bash
go work init ./protos ./bookstore
```

---

## 🧠 Summary

- Protobufs live in `protos/` and are imported across services.
- `bookstore/` follows **Clean Architecture** with domain, service, adapter split.
- Uses gRPC with global error handling and modular layering.
