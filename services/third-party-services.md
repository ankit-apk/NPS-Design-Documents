## 📘 Service Overview

**Third Party Integration Service** acts as a central proxy and abstraction layer for all third-party systems (Aadhaar, PAN, payment, SMS, email). It standardizes request/response formats, handles retries, logs, and translates third-party errors into domain errors.

---

## 🏛️ Project Structure

```
/thirdparty
│
├── api/
│   └── thirdparty.proto
│
├── cmd/
│   └── server/
│       └── main.go
│
├── internal/
│   └── thirdparty/
│       ├── handler/
│       ├── service/
│       ├── repo/
│       └── model.go
│
├── pkg/
│   ├── errors/
│   ├── logger/
│   └── httpclient/       # for HTTP/REST-based third party calls
│
├── test/
└── go.mod
```

---

## 🔌 gRPC Proto Definitions

```proto
// api/thirdparty.proto

syntax = "proto3";

option go_package = "thirdparty/api";

package thirdparty;

// Aadhaar Verification
service ThirdPartyService {
  rpc VerifyAadhaar(VerifyAadhaarRequest) returns (VerifyAadhaarResponse);
  rpc VerifyPAN(VerifyPANRequest) returns (VerifyPANResponse);
  rpc SendSMS(SendSMSRequest) returns (SendSMSResponse);
  rpc SendEmail(SendEmailRequest) returns (SendEmailResponse);
  rpc InitiatePayment(PaymentRequest) returns (PaymentResponse);
}

message VerifyAadhaarRequest {
  string aadhaar_number = 1; // Aadhaar number to verify
}

message VerifyAadhaarResponse {
  bool verified = 1;
  string name = 2;
  string dob = 3;
}

message VerifyPANRequest {
  string pan_number = 1;
}

message VerifyPANResponse {
  bool valid = 1;
  string name = 2;
}

message SendSMSRequest {
  string phone = 1;
  string message = 2;
}

message SendSMSResponse {
  string status = 1;
}

message SendEmailRequest {
  string to = 1;
  string subject = 2;
  string body = 3;
}

message SendEmailResponse {
  string status = 1;
}

message PaymentRequest {
  double amount = 1;
  string user_id = 2;
}

message PaymentResponse {
  string transaction_id = 1;
  string status = 2;
}
```

---

## 🧠 Domain Model (`internal/thirdparty/model.go`)

```go
type AadhaarVerification struct {
	AadhaarNumber string
	Name          string
	DOB           string
	Verified      bool
}

type PANVerification struct {
	PANNumber string
	Name      string
	Valid     bool
}
```

---

## 🧩 Repository Layer

```go
// internal/thirdparty/repo/interface.go

type ThirdPartyRepository interface {
	VerifyAadhaar(ctx context.Context, aadhaar string) (*model.AadhaarVerification, error)
	VerifyPAN(ctx context.Context, pan string) (*model.PANVerification, error)
	SendSMS(ctx context.Context, phone, message string) error
	SendEmail(ctx context.Context, to, subject, body string) error
	ProcessPayment(ctx context.Context, userID string, amount float64) (string, error)
}
```

Implementations:

- `aadhaarClient.go`
- `panClient.go`
- `emailClient.go`
- `smsClient.go`
- `paymentClient.go`

---

## 🧪 Service Layer

```go
// internal/thirdparty/service/service.go

type Service struct {
	repo repo.ThirdPartyRepository
}

func New(repo repo.ThirdPartyRepository) *Service {
	return &Service{repo: repo}
}

func (s *Service) VerifyAadhaar(ctx context.Context, aadhaar string) (*model.AadhaarVerification, error) {
	if len(aadhaar) != 12 {
		return nil, errors.ErrInvalidAadhaar
	}
	return s.repo.VerifyAadhaar(ctx, aadhaar)
}

// Other methods follow similar structure...
```

---

## 🧑‍💻 Handler Layer

```go
// internal/thirdparty/handler/handler.go

type Handler struct {
	svc *service.Service
}

func New(svc *service.Service) *Handler {
	return &Handler{svc: svc}
}

func (h *Handler) VerifyAadhaar(ctx context.Context, req *pb.VerifyAadhaarRequest) (*pb.VerifyAadhaarResponse, error) {
	result, err := h.svc.VerifyAadhaar(ctx, req.AadhaarNumber)
	if err != nil {
		return nil, errors.ToGRPCError(err)
	}
	return &pb.VerifyAadhaarResponse{
		Verified: result.Verified,
		Name:     result.Name,
		Dob:      result.DOB,
	}, nil
}
```

---

## 🚨 Error Handling

```go
// pkg/errors/errors.go

var ErrInvalidAadhaar = &AppError{
	Code:   "INVALID_AADHAAR",
	Reason: "INVALID_FORMAT",
	Message: "Aadhaar must be 12 digits",
	GRPC:   codes.InvalidArgument,
}
```

---

## 🧵 Dependency Injection (`cmd/server/main.go`)

```go
func main() {
	logger := logger.New()

	repo := thirdparty.NewHTTPRepo() // implements ThirdPartyRepository
	svc := service.New(repo)
	h := handler.New(svc)

	grpcServer := grpc.NewServer()
	pb.RegisterThirdPartyServiceServer(grpcServer, h)

	lis, _ := net.Listen("tcp", ":50051")
	logger.Info("gRPC server listening...")
	grpcServer.Serve(lis)
}
```

---

## 📈 Logging

Use `pkg/logger` for logging requests/responses **only in the handler layer**, e.g.:

```go
logger.Infof("Sending SMS to %s", phone)
```

PII should be masked or avoided in logs.

---

## ✅ Testing Strategy

- **Unit Tests**:

  - Mock `ThirdPartyRepository`
  - Test `Service` layer using table-driven tests

- **Integration Tests**:

  - Test `Handler` layer with gRPC server in `test/`

---

## 🚀 Future Extensibility

Add new providers:

1. Add method in `ThirdPartyRepository`
2. Implement in repo
3. Update service & handler
4. Update proto definitions

Example: Add `VerifyDrivingLicense`, etc.
