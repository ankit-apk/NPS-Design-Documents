# Low-Level Design Document: NPS Subscriber SA/PE/IC Withdrawal Service

**Version:** 1.0
**Date:** October 26, 2023
**Author:** AI Assistant

## Table of Contents

1.  [Introduction](#1-introduction)
    1.1. [Purpose of the System Software](#11-purpose-of-the-system-software)
    1.2. [Scope of the LLD Document](#12-scope-of-the-lld-document)
    1.3. [References](#13-references)
2.  [System Architecture Overview](#2-system-architecture-overview)
    2.1. [Description](#21-description)
    2.2. [Service Interactions Diagram](#22-service-interactions-diagram)
    2.3. [Architectural Style](#23-architectural-style)
    2.4. [Inter-Service Communication Strategy](#24-inter-service-communication-strategy)
    2.5. [Role of Key Infrastructure Services](#25-role-of-key-infrastructure-services)
    2.5.1. [Temporal.io (Workflow Orchestration)](#251-temporalio-workflow-orchestration)
    2.5.2. [Apache Kafka (Messaging Service)](#252-apache-kafka-messaging-service)
    2.5.3. [Redis (Caching Service)](#253-redis-caching-service)
3.  [Component Designs](#3-component-designs)
    3.1. [Overview of Core Services/Components](#31-overview-of-core-servicescomponents)
    3.1.1. [Withdrawal Orchestration Service (Leveraging Temporal.io)](#311-withdrawal-orchestration-service-leveraging-temporalio)
    3.1.2. [User Service Integration](#312-user-service-integration)
    3.1.3. [Receipt Service](#313-receipt-service)
    3.1.4. [NPS Corpus & Eligibility Evaluation Engine (Rule Engine)](#314-nps-corpus--eligibility-evaluation-engine-rule-engine)
    3.1.5. [Fund Manager Service Integration](#315-fund-manager-service-integration)
    3.1.6. [Nominee Management Component](#316-nominee-management-component)
    3.1.7. [Bank Account Verification Module](#317-bank-account-verification-module)
    3.1.8. [Document Management Service](#318-document-management-service)
    3.1.9. [Notification Service](#319-notification-service)
    3.1.10. [Acknowledgement Service](#3110-acknowledgement-service)
    3.1.11. [CRA Gateway Service](#3111-cra-gateway-service)
    3.1.12. [Caching Abstraction Layer](#3112-caching-abstraction-layer)
    3.2. [API Interfaces](#32-api-interfaces)
    3.2.1. [Error Code Strategy](#321-error-code-strategy)
    3.2.2. [Provided Interfaces (Withdrawal Service Facade)](#322-provided-interfaces-withdrawal-service-facade)
    3.2.3. [Required Interfaces (Dependencies)](#323-required-interfaces-dependencies)
    3.3. [Temporal Workflow Design Examples](#33-temporal-workflow-design-examples)
    3.3.1. [Superannuation/Premature/Incapacitation Withdrawal Workflow](#331-superannuationprematureincapacitation-withdrawal-workflow)
4.  [System Initialization](#4-system-initialization)
5.  [Data Management](#5-data-management)
    5.1. [Data Models (Key Entities)](#51-data-models-key-entities)
    5.2. [Data Integrity and Security](#52-data-integrity-and-security)
6.  [Appendices](#6-appendices)
    6.1. [Eligibility Evaluation Logic (Illustrative)](#61-eligibility-evaluation-logic-illustrative)
    6.2. [Diagram Stubs](#62-diagram-stubs)
    6.3. [Glossary](#63-glossary)
7.  [Notes on Usage](#7-notes-on-usage)
    7.1. [Customization](#71-customization)
    7.2. [Focus of the LLD](#72-focus-of-the-lld)

---

## 1. Introduction

### 1.1. Purpose of the System Software

The purpose of this system is to enable a streamlined, secure, and resilient withdrawal process from the National Pension System (NPS) under the categories of **Superannuation (SA)**, **Premature Exit (PE)**, and **Incapacitation (IC)**. It orchestrates interactions with several critical microservices, including user profile validation, corpus eligibility assessment, annuity plan selection, document verification, bank account validation, and final consent through e-signature or OTP. The system is designed to ensure full compliance with PFRDA regulatory guidelines and CRA (Central Recordkeeping Agency) requirements, while offering a user-friendly and guided experience. This LLD focuses on the backend services and orchestration logic.

### 1.2. Scope of the LLD Document

This Low-Level Design (LLD) document details the internal design, technical components, APIs, and workflows of the NPS Withdrawal Orchestration System for Superannuation, Premature Exit, and Incapacitation flows. It covers:

- Selection and processing of withdrawal categories (SA, PE, IC).
- Orchestration of interactions with dependent microservices (User, Receipt, Rule Engine, Annuity, Payment Gateway, Document, Notification, CRA).
- Integration with Temporal.io for robust workflow management.
- Utilization of Kafka for asynchronous eventing and Redis for caching.
- Detailed API specifications for internal and external communication.
- Data models for key entities.
- Error handling, security considerations, and data integrity measures.
- System initialization and operational considerations.

This LLD explicitly addresses:

- Modernization of the NPS withdrawal infrastructure using scalable, resilient, and secure technologies.
- Data protection and privacy, ensuring secure storage and transmission of sensitive data.
- API security, regulatory compliance (PFRDA, CRA mandates), and robust access control.

### 1.3. References

- User Service API Documentation
- Annuity Service API Documentation
- Notification Service (SMS/OTP/Email) Documentation
- Document Handling & CRA Integration Guidelines
- Penny-drop Payment Gateway Integration Docs
- CRA Compliance Protocols for Exit Management
- PFRDA Master Circulars on Exits & Withdrawals
- Temporal.io Documentation
- Apache Kafka Documentation
- Redis Documentation

---

## 2. System Architecture Overview

### 2.1. Description

The NPS Withdrawal System employs a microservices architecture, with the **Withdrawal Orchestration Service** (built using Temporal.io) at its core. This service manages the end-to-end withdrawal process for SA, PE, and IC cases by coordinating calls to various specialized domain services. User-facing interactions are exposed via an API Gateway, which routes requests to the appropriate backend services or initiates Temporal workflows.

### 2.2. Service Interactions Diagram

_(This section would ideally contain a high-level diagram illustrating the services and their primary interactions. See [Appendix 6.2](#62-diagram-stubs) for description.)_

Key interactions include:

1.  **User Request Initiation:** Frontend/PoP systems initiate withdrawal requests via an API Gateway.
2.  **Workflow Trigger:** The API Gateway (or a dedicated facade service) triggers a Temporal workflow in the Withdrawal Orchestration Service corresponding to the withdrawal type (SA/PE/IC).
3.  **Data Fetching & Validation:** The workflow orchestrates calls to:
    - `User Service`: Fetches subscriber profile, nominee, and bank details.
    - `Receipt Service`: Retrieves/generates Claim ID.
    - `Rule Engine Service`: Validates eligibility, calculates corpus, and applies withdrawal rules.
    - `Caching Service (Redis)`: Queried for frequently accessed, less volatile data.
4.  **User Choices & Input:**
    - `Annuity Service`: Fetches ASPs and schemes; user selection is captured.
    - `Document Service`: Handles document uploads and validation.
5.  **Verifications:**
    - `Payment Gateway Service`: Verifies bank account via Penny Drop.
    - `Notification Service`: Sends OTP for various verification steps.
6.  **Consent & Submission:**
    - Workflow manages e-sign or OTP-based consent.
    - Bundled data and documents are submitted to the `CRA Gateway Service`.
7.  **Asynchronous Events:** `Kafka` is used for publishing significant domain events (e.g., `WithdrawalSubmitted`, `DocumentVerified`) for consumption by other interested services (auditing, analytics, etc.).
8.  **Acknowledgement:** `Receipt Service` (or a dedicated `Acknowledgement Service`) generates and provides an acknowledgement ID.

### 2.3. Architectural Style

- **Microservices Architecture:** The system is composed of small, independent, and deployable services, each responsible for a specific business capability.
- **Domain-Driven Design (DDD):** Services are designed around Bounded Contexts (e.g., Subscriber Management, Withdrawal Processing, Document Handling).
- **Orchestration over Choreography (for core withdrawal flow):** Temporal.io is used to explicitly define and manage the multi-step withdrawal process, ensuring reliability, visibility, and easier handling of failures and compensations (Saga pattern).
- **Event-Driven Aspects:** Kafka is used for notifying other systems or triggering decoupled processes based on events occurring within the withdrawal workflow.

### 2.4. Inter-Service Communication Strategy

A hybrid approach is used:

- **Synchronous Communication:**
  - **gRPC:** Preferred for high-performance, low-latency internal service-to-service communication where contracts are well-defined (e.g., calls from Temporal activities to specific microservices). Benefits include Protobuf efficiency and strong typing.
  - **RESTful APIs (HTTPS/JSON):** Used for external-facing APIs (API Gateway to clients) and for services where broader compatibility or simpler integration is desired.
- **Asynchronous/Event-Driven Communication:**
  - **Temporal.io Workflows & Activities:** For orchestrating the main withdrawal logic, including retries, timeouts, and sagas.
  - **Apache Kafka:** Used as a message broker for:
    - **Event Notification:** Services publish domain events (e.g., `WithdrawalRequestInitiated`, `CRAsubmissionSuccessful`). Other services (e.g., auditing, analytics, downstream systems) can subscribe to these events.
    - **Decoupling:** For tasks that don't need immediate synchronous response within the core workflow.
    - **Load Leveling:** Can help absorb spikes for certain non-critical asynchronous tasks.

### 2.5. Role of Key Infrastructure Services

#### 2.5.1. Temporal.io (Workflow Orchestration)

- **Responsibility:** Manages the state and progression of long-running withdrawal requests (SA, PE, IC).
- **Functionality:**
  - Defines withdrawal workflows as durable, stateful functions.
  - Executes individual steps as "Activities" (which are calls to microservices).
  - Handles automatic retries of activities on transient failures.
  - Manages timeouts for activities and entire workflows.
  - Facilitates implementation of the Saga pattern for distributed transactions (e.g., ensuring cleanup actions if a step fails after prior steps have committed changes).
  - Provides visibility into ongoing and completed workflows.
- **Impact:** Significantly improves the reliability and resilience of the complex withdrawal process compared to manual orchestration or simple choreography.

#### 2.5.2. Apache Kafka (Messaging Service)

- **Responsibility:** Provides a distributed, fault-tolerant event streaming platform.
- **Functionality:**
  - Serves as a message bus for domain events.
  - Enables decoupling of services; producers don't need to know about consumers.
  - Supports at-least-once/exactly-once semantics for event delivery (with proper configuration).
- **Use Cases:**
  - Broadcasting `WithdrawalStatusChanged` events.
  - Ingesting data for an audit trail service.
  - Triggering asynchronous notifications not directly managed by the core workflow.

#### 2.5.3. Redis (Caching Service)

- **Responsibility:** Provides a fast, in-memory data store for caching.
- **Functionality:**
  - Stores frequently accessed, relatively static data to reduce latency and load on primary data sources.
  - Can be used for session management (e.g., short-lived UI state tokens before a workflow starts).
  - Supports various data structures (strings, hashes, lists).
- **Use Cases:**
  - Caching ASP master lists and their schemes.
  - Caching user profile data if it's read multiple times during a session.
  - Storing results of certain rule engine evaluations if inputs are identical and rules don't change frequently.
  - Rate limiting information.

---

## 3. Component Designs

### 3.1. Overview of Core Services/Components

#### 3.1.1. Withdrawal Orchestration Service (Leveraging Temporal.io)

- **Responsibilities:**
  - Manages the end-to-end lifecycle of a withdrawal request (SA, PE, IC) using Temporal workflows.
  - Initiates and coordinates activities across various microservices based on the defined workflow logic.
  - Handles user choices, validations, document processing steps, consent, and CRA submission within the workflow.
  - Implements compensation logic (Sagas) for failed workflows.
- **Key Workflows (Examples):**
  - `SuperannuationWithdrawalWorkflow`
  - `PrematureExitWithdrawalWorkflow`
  - `IncapacitationWithdrawalWorkflow`
- **Key Activities (Examples, invoked by workflows):**
  - `FetchUserDetailsActivity`
  - `GenerateClaimIdActivity`
  - `EvaluateEligibilityActivity`
  - `FetchAnnuityProvidersActivity`
  - `VerifyBankAccountActivity`
  - `ProcessDocumentUploadActivity`
  - `SendOtpActivity`
  - `VerifyOtpActivity`
  - `SubmitToCraActivity`
  - `GenerateAcknowledgementActivity`

#### 3.1.2. User Service Integration

- **Responsibilities (as a dependency):**
  - Provides subscriber profile data (personal details, contact info, marital status).
  - Provides nominee details.
  - Provides registered bank account details.
- **Interaction:** Called by Temporal activities (e.g., `FetchUserDetailsActivity`) within the withdrawal workflow. Data is cached by the Caching Abstraction Layer where appropriate.

#### 3.1.3. Receipt Service

- **Responsibilities:**
  - Generates or retrieves unique **Claim IDs**.
    - **SA:** Claim ID might be pre-generated (e.g., 6 months prior to retirement). Service retrieves this.
    - **PE/IC:** Claim ID is generated upon initiation of the withdrawal workflow. The service ensures uniqueness and proper formatting.
  - Generates **Acknowledgement IDs** upon successful completion of certain stages (e.g., final submission).
- **Interaction:** Called by Temporal activities (e.g., `GenerateClaimIdActivity`, `GenerateAcknowledgementActivity`).

#### 3.1.4. NPS Corpus & Eligibility Evaluation Engine (Rule Engine)

- **Responsibilities:**
  - Retrieves current corpus balance (interfacing with a service that has this data, or taking it as input).
  - Evaluates withdrawal eligibility based on type (SA, PE, IC), corpus amount, age, contribution period, and other PFRDA rules.
  - Determines permissible withdrawal percentages (lump-sum vs. annuity).
- **Illustrative Rules (to be verified with actual PFRDA guidelines):**
  - **SA:**
    - Corpus ≤ ₹5 lakh: 100% withdrawal allowed.
    - Corpus > ₹5 lakh: Max 60% withdrawal, min 40% annuitization.
    - Age: Must have reached superannuation age as per NPS rules.
  - **PE:**
    - Corpus ≤ ₹2.5 lakh: 100% withdrawal allowed.
    - Corpus > ₹2.5 lakh: Max 20% withdrawal, min 80% annuitization.
    - Conditions: Minimum vesting period (e.g., 3 years, 10 years for specific cases), specific reasons for exit.
  - **IC (Incapacitation):**
    - May allow 100% withdrawal irrespective of corpus amount (subject to PFRDA rules).
    - Requires valid incapacitation proof.
    - No minimum service period might be required.
- **Interaction:** Called by a Temporal activity (e.g., `EvaluateEligibilityActivity`). Rules might be configurable and potentially cached.

#### 3.1.5. Fund Manager / Annuity Service Integration

- **Responsibilities (as a dependency):**
  - Provides a list of available Annuity Service Providers (ASPs) and their schemes.
  - Filters schemes based on withdrawal type, marital status, and other criteria.
- **Interaction:** Called by a Temporal activity (e.g., `FetchAnnuityProvidersActivity`). ASP and scheme data is a good candidate for caching. The workflow captures the user's annuity selection.

#### 3.1.6. Nominee Management Component

- **Responsibilities (within User Service or as a separate component called by the workflow):**
  - Provides existing nominee details.
  - Allows modification/addition of nominees if permitted by the flow and regulations.
- **Interaction:** Data fetched by `FetchUserDetailsActivity`. Updates to nominees (if allowed) would involve a dedicated activity and service call.

#### 3.1.7. Bank Account Verification Module

- **Responsibilities (typically part of a Payment Gateway Service):**
  - Verifies bank account status and beneficiary name using Penny Drop mechanism.
- **Interaction:** Called by a Temporal activity (e.g., `VerifyBankAccountActivity`). The workflow proceeds only upon successful verification.

#### 3.1.8. Document Management Service

- **Responsibilities:**
  - Allows users to upload required documents based on withdrawal type (SA, PE, IC).
    - **SA:** Retirement proof, PAN, address proof, bank proof.
    - **PE:** Resignation letter/relevant proof, PAN, address proof, bank proof.
    - **IC:** Medical certificate of incapacitation from a designated authority, PAN, address proof, bank proof.
  - Validates documents for type (PDF, JPG, PNG), size, and basic format.
  - **File Integrity:** Generates and stores checksums (e.g., SHA-256) for each uploaded file. Verifies checksums upon retrieval.
  - Bundles all validated documents into a single package (e.g., a ZIP file or a merged PDF with a manifest) for CRA submission. May apply digital signatures to the bundle if required by CRA.
- **Interaction:** Users upload via a UI that calls this service. A Temporal activity (e.g., `ProcessDocumentUploadActivity`) coordinates the upload, validation, and bundling status.

#### 3.1.9. Notification Service

- **Responsibilities:**
  - Sends OTPs (SMS, Email) for various verification steps (e.g., mobile verification, pre-submission consent).
  - Sends acknowledgements and status updates (Email, SMS) at different stages of the withdrawal process.
  - Manages OTP generation, sending, and validation requests (validation logic might reside here or be a separate OTP service).
- **Email Security:** For emails, ensures the sending domain is configured with SPF, DKIM, and DMARC records to improve deliverability and prevent spoofing.
- **Interaction:** Called by Temporal activities (e.g., `SendOtpActivity`, `VerifyOtpActivity`, `SendAcknowledgementEmailActivity`).

#### 3.1.10. Acknowledgement Service

- **Responsibilities:** (May be part of Receipt Service or a distinct service)
  - Generates a comprehensive acknowledgement receipt/document for the subscriber upon successful submission to CRA.
  - Includes Claim ID, Acknowledgement ID, summary of choices, and submission timestamp.
- **Interaction:** Called by a Temporal activity at the end of a successful workflow.

#### 3.1.11. CRA Gateway Service

- **Responsibilities:**
  - Acts as an Anti-Corruption Layer (ACL) to the external CRA system.
  - Formats the withdrawal request data and document bundle according to CRA specifications.
  - Submits the request to the CRA platform.
  - Handles responses and error codes from CRA, translating them for internal use.
  - May handle asynchronous CRA status updates if the CRA process is not immediate.
- **Interaction:** Called by a Temporal activity (e.g., `SubmitToCraActivity`).

#### 3.1.12. Caching Abstraction Layer

- **Responsibilities:**
  - Provides a common interface for services to interact with Redis (or another caching provider).
  - Implements caching strategies (e.g., read-through, write-through, cache-aside).
  - Manages cache keys, TTLs, and invalidation logic.
- **Interaction:** Used by various services and Temporal activities to store and retrieve cached data.

### 3.2. API Interfaces

#### 3.2.1. Error Code Strategy

A standardized error code strategy will be used.

| Category          | HTTP Status Code | Internal Error Code Prefix | Example Description                      |
| ----------------- | ---------------- | -------------------------- | ---------------------------------------- |
| Validation Error  | 400              | `VAL_`                     | `VAL_001: Invalid PRAN ID format`        |
|                   |                  |                            | `VAL_002: Missing required document`     |
| Authentication    | 401              | `AUTH_`                    | `AUTH_001: Invalid or expired token`     |
| Authorization     | 403              | `AUTZ_`                    | `AUTZ_001: User not eligible for action` |
| Not Found         | 404              | `NF_`                      | `NF_001: Subscriber details not found`   |
| Conflict          | 409              | `CNFL_`                    | `CNFL_001: Withdrawal already initiated` |
| External Service  | 502 / 504        | `EXT_`                     | `EXT_CRA_001: CRA system timeout`        |
|                   |                  |                            | `EXT_BANK_002: Bank verification failed` |
| Internal Server   | 500              | `INT_`                     | `INT_001: Unhandled internal error`      |
| Temporal Workflow | 500 / specific   | `WF_`                      | `WF_001: Workflow execution failed`      |

_Specific CRA error codes will be mapped and exposed with consistent internal prefixes._

#### 3.2.2. Provided Interfaces (Withdrawal Service Facade / API Gateway Endpoints)

These are illustrative. The actual initiation might be a single endpoint that then starts a specific Temporal workflow based on `withdrawalCategory`.

**1. Initiate Withdrawal Workflow**

- **Endpoint:** `POST /api/v1/withdrawals`
- **Headers:** `Authorization: Bearer <Token>`, `Content-Type: application/json`, `Idempotency-Key: <UUID>`
- **Purpose:** To initiate a new withdrawal workflow (SA, PE, or IC) for a subscriber.
- **Request Body:**
  ```json
  {
    "pran": "123456789012", // Subscriber PRAN ID
    "withdrawalCategory": "Superannuation", // Enum: "Superannuation", "PrematureExit", "Incapacitation"
    "withdrawalReason": "Retirement" // Optional, more specific reason if applicable
    // Potentially other initial parameters if known upfront
  }
  ```
- **Success Response (202 Accepted):** Indicates workflow has been successfully started.
  ```json
  {
    "status": "success",
    "message": "Withdrawal workflow initiated successfully.",
    "workflowId": "temporal_workflow_uuid_123", // Temporal Workflow ID for tracking
    "pran": "123456789012",
    "withdrawalCategory": "Superannuation"
  }
  ```
- **Error Responses:** 400 (Validation), 401 (Auth), 403 (Forbidden), 409 (Conflict if already active), 500.

**2. Get Withdrawal Workflow Status**

- **Endpoint:** `GET /api/v1/withdrawals/{workflowId}/status`
- **Headers:** `Authorization: Bearer <Token>`
- **Purpose:** To get the current status of an ongoing or completed withdrawal workflow.
- **Success Response (200 OK):**
  ```json
  {
    "workflowId": "temporal_workflow_uuid_123",
    "pran": "123456789012",
    "withdrawalCategory": "Superannuation",
    "status": "InProgress", // Or "Completed", "Failed", "TimedOut", "Cancelled"
    "currentStep": "AwaitingBankVerification", // Human-readable current step
    "createdAt": "2023-10-26T10:00:00Z",
    "updatedAt": "2023-10-26T12:30:00Z",
    "details": {
      // Optional: more specific details based on status
      "failureReason": null // Populated if status is "Failed"
    }
  }
  ```
- **Error Responses:** 401, 403, 404 (Not Found).

**3. Signal Withdrawal Workflow (e.g., User provides input)**

- Temporal workflows can receive signals. This is an abstraction. The actual API might be specific to the data being provided.
- **Example: Submit Annuity Selection**
  - **Endpoint:** `POST /api/v1/withdrawals/{workflowId}/annuity-selection`
  - **Headers:** `Authorization: Bearer <Token>`, `Content-Type: application/json`
  - **Request Body:**
    ```json
    {
      "aspId": "ASP001",
      "schemeId": "SCHM001_A",
      "frequency": "Monthly"
      // Other annuity related details
    }
    ```
  - **Success Response (200 OK):**
    ```json
    {
      "status": "success",
      "message": "Annuity selection received and processed."
    }
    ```
- **Error Responses:** 400, 401, 403, 404.

**4. List User's Withdrawal Requests**

- **Endpoint:** `GET /api/v1/subscriber/withdrawals` (Gets withdrawals for the authenticated user/PRAN from token)
- **Headers:** `Authorization: Bearer <Token>`
- **Query Parameters:** `status=InProgress` (optional), `category=Superannuation` (optional), `page=1`, `pageSize=10`
- **Success Response (200 OK):**
  ```json
  {
    "status": "success",
    "data": [
      {
        "workflowId": "temporal_workflow_uuid_123",
        "pran": "123456789012",
        "withdrawalCategory": "Superannuation",
        "status": "InProgress",
        "createdAt": "2023-10-26T10:00:00Z"
      },
      {
        "workflowId": "temporal_workflow_uuid_456",
        "pran": "123456789012",
        "withdrawalCategory": "PrematureExit",
        "status": "Completed",
        "createdAt": "2023-09-15T14:00:00Z"
      }
    ],
    "pagination": {
      "currentPage": 1,
      "pageSize": 10,
      "totalItems": 2,
      "totalPages": 1
    }
  }
  ```

_(Note: The original LLD had many GET/POST endpoints for each step like save-data, get-form-details. With Temporal, many of these become signals or queries to the workflow, or data passed between activities. The frontend would interact with the workflow through a more limited set of facade APIs or query workflow state.)_

**The provided APIs from the original LLD like `GET /withdrawal-types`, `GET /annuity-providers` would still exist but likely be provided by dedicated facade services or the API Gateway, which in turn call the relevant backend microservices (not directly part of the withdrawal orchestration component, but consumed by it or the UI).**

#### 3.2.3. Required Interfaces (Dependencies - Examples of gRPC Service Definitions used by Temporal Activities)

**1. User Service (gRPC)**

```protobuf
service UserService {
  rpc GetSubscriberDetails(GetSubscriberDetailsRequest) returns (GetSubscriberDetailsResponse);
  rpc GetNomineeDetails(GetNomineeDetailsRequest) returns (GetNomineeDetailsResponse);
  rpc GetBankDetails(GetBankDetailsRequest) returns (GetBankDetailsResponse);
}

message GetSubscriberDetailsRequest {
  string pran_id = 1;
}

message GetSubscriberDetailsResponse {
  string pran_id = 1;
  string name = 2;
  string date_of_birth = 3; // YYYY-MM-DD
  string marital_status = 4; // "Married", "Unmarried", "Divorced", "Widowed"
  // ... other fields
  BankDetails bank_details = 10;
  repeated Nominee nominees = 11;
}
// Define Nominee, BankDetails messages
```

**2. Rule Engine Service (gRPC)**

```protobuf
service RuleEngineService {
  rpc EvaluateWithdrawalEligibility(EvaluateEligibilityRequest) returns (EvaluateEligibilityResponse);
}

message EvaluateEligibilityRequest {
  string pran_id = 1;
  string withdrawal_category = 2; // "Superannuation", "PrematureExit", "Incapacitation"
  double corpus_amount_tier1 = 3;
  double corpus_amount_tier2 = 4; // If applicable
  string date_of_birth = 5; // YYYY-MM-DD
  string date_of_joining_nps = 6; // YYYY-MM-DD
  // ... other relevant parameters for rules
}

message EvaluateEligibilityResponse {
  bool eligible = 1;
  string reason_code = 2; // If not eligible
  string reason_message = 3;
  double permissible_lumpsum_percentage = 4;
  double mandatory_annuity_percentage = 5;
  // ... other rule outcomes
}
```

**3. Document Service (gRPC/REST)**

- `UploadDocument`: (Likely REST for multipart/form-data from client, then internal processing)
- `GetDocumentStatus`:
- `BundleDocumentsForCRA`:

**4. Payment Gateway Service (Penny Drop) (gRPC/REST)**

```protobuf
service PaymentGatewayService {
  rpc VerifyBankAccount(BankAccountVerificationRequest) returns (BankAccountVerificationResponse);
}

message BankAccountVerificationRequest {
  string pran_id = 1;
  string account_holder_name = 2;
  string account_number = 3;
  string ifsc_code = 4;
}

message BankAccountVerificationResponse {
  string verification_id = 1;
  string status = 2; // "Verified", "Failed", "Pending"
  string message = 3;
  string verified_account_holder_name = 4; // Name as per bank records
}
```

_(Response for other services like Annuity, Notification, CRA Gateway would be similarly defined.)_

### 3.3. Temporal Workflow Design Examples

#### 3.3.1. Superannuation/Premature/Incapacitation Withdrawal Workflow

This is a conceptual outline. Each workflow type (SA, PE, IC) would have its own specific variations.

**Workflow Definition (Conceptual Pseudocode):**

```
@WorkflowInterface
public interface NPSWithdrawalWorkflow {
    @WorkflowMethod
    String processWithdrawal(WithdrawalRequestInputs inputs);

    @SignalMethod
    void provideAnnuitySelection(AnnuityDetails details);

    @SignalMethod
    void provideDocumentsUploaded(List<DocumentInfo> documents);

    @SignalMethod
    void provideOtpForConsent(String otp);

    @QueryMethod
    WorkflowStatus getStatus();
}

public class NPSWithdrawalWorkflowImpl implements NPSWithdrawalWorkflow {
    // Activities (stubs for actual microservice calls)
    private final UserActivities userActivities = Workflow.newActivityStub(UserActivities.class, ...);
    private final RuleEngineActivities ruleEngineActivities = Workflow.newActivityStub(...);
    // ... other activities

    public String processWithdrawal(WithdrawalRequestInputs inputs) {
        // 1. Fetch User Details (Profile, Bank, Nominees)
        UserDetails user = userActivities.fetchUserDetails(inputs.getPran());
        // Store in workflow state or pass around

        // 2. Generate/Fetch Claim ID
        String claimId = receiptActivities.generateClaimId(inputs.getPran(), inputs.getCategory());

        // 3. Evaluate Eligibility (SA/PE/IC specific logic)
        EligibilityResult eligibility = ruleEngineActivities.evaluateEligibility(user, inputs.getCategory(), corpus);
        if (!eligibility.isEligible()) {
            // Compensate if needed, end workflow as failed
            return "Failed: Not Eligible - " + eligibility.getReason();
        }

        // 4. Bank Account Penny Drop Verification
        // Retry options can be configured for this activity
        BankAccountVerificationResult bankResult = paymentActivities.verifyBankAccount(user.getBankDetails());
        if (!bankResult.isVerified()) {
            // Handle failure, perhaps wait for user to update bank details (another signal)
            return "Failed: Bank Verification Failed";
        }

        // 5. Annuity Selection (if applicable, e.g., not for 100% lump sum)
        if (eligibility.requiresAnnuity()) {
            List<AnnuityProvider> providers = annuityActivities.getAnnuityProviders(inputs.getCategory(), user.getMaritalStatus());
            // Signal UI to show options. Workflow waits for provideAnnuitySelection signal.
            Workflow.await(() -> this.annuitySelectionReceived);
        }

        // 6. Document Upload & Verification
        // Signal UI for document upload. Workflow waits for provideDocumentsUploaded signal.
        Workflow.await(() -> this.documentsConfirmed);
        DocumentBundle bundle = documentActivities.bundleDocuments(this.uploadedDocuments);


        // 7. User Consent (OTP / E-sign)
        String consentOtp = notificationActivities.sendConsentOtp(user.getMobile());
        Workflow.await(() -> this.consentOtpProvided); // Wait for signal
        boolean otpValid = notificationActivities.verifyOtp(consentOtp, this.receivedOtp);
        if (!otpValid) return "Failed: Consent OTP Invalid";

        // 8. Submit to CRA
        // Implements Saga pattern for compensation if CRA submission fails
        try {
            CRAsubmissionResult craResult = craActivities.submitToCRA(claimId, user, bundle, this.annuitySelection);
            if (!craResult.isSuccess()) {
                 // Log, potentially trigger manual intervention alert
                return "Failed: CRA Submission Error - " + craResult.getErrorMessage();
            }
        } catch (ActivityFailure e) {
            // Handle retries exhaust / non-retryable error
            // Compensate previous steps if necessary (e.g. mark ClaimID as void)
            return "Failed: CRA Submission Activity Failed";
        }


        // 9. Generate Acknowledgement
        Acknowledgement ack = receiptActivities.generateAcknowledgement(claimId, craResult.getCraAckId());
        notificationActivities.sendAcknowledgement(user, ack);

        return "Success: " + ack.getAcknowledgementId();
    }
    // ... signal handlers and query method implementation
}
```

**Key considerations within Temporal workflows:**

- **Idempotency:** Activities should be idempotent.
- **Timeouts & Retries:** Configure appropriate timeouts (schedule-to-start, start-to-close) and retry policies for activities.
- **Saga Pattern:** For multi-step transactions that span services, use Saga for compensation. Temporal makes this easier by reliably executing compensation activities if a forward activity fails.
- **State Management:** Workflow state is maintained by Temporal and should be kept reasonably small. Large data objects are typically passed by reference (e.g., document IDs) or handled by activities.
- **Versioning:** Temporal supports workflow versioning for handling changes to workflow logic over time.

---

## 4. System Initialization

The startup process involves:

1.  **Infrastructure Boot:**
    - Core system resources (compute, network).
    - Environment-specific configurations (from .env, secret managers).
2.  **Dependent Infrastructure Services:**
    - **Database Services (PostgreSQL, YugabyteDB, etc.):** Start, run migrations, seed master data.
    - **Redis Cluster:** Ensure nodes are up and clustered.
    - **Apache Kafka Cluster & Zookeeper:** Start brokers, ensure topics are created with correct partitions and replication factors (e.g., `withdrawal-events`, `audit-log-events`).
    - **Temporal.io Cluster:**
      - Start Temporal server (frontend, history, matching, worker services).
      - Ensure necessary namespaces are registered.
3.  **Microservices Deployment & Initialization:**
    - Deploy individual microservices in a defined order or ensure readiness probes are effective.
    - Services connect to databases, Kafka, Redis.
    - **Temporal Workers:** Deploy worker services that poll specific task queues for the defined workflows and activities. These workers host the workflow and activity implementations.
    - Register workflow and activity implementations with the Temporal client.
4.  **API Gateway:** Configure routes to the facade services or appropriate backend entry points.
5.  **Health Checks:** All services should expose health check endpoints. Kubernetes or other orchestrators use these for readiness and liveness probes.

---

## 5. Data Management

### 5.1. Data Models (Key Entities - illustrative, actual persistence is within domain services)

**WithdrawalRequest (Conceptual - state largely managed by Temporal)**

- `workflowId` (String, PK - Temporal ID)
- `pran` (String, Index)
- `claimId` (String, Unique, Index)
- `withdrawalCategory` (Enum: SA, PE, IC)
- `withdrawalReason` (String)
- `status` (Enum: INITIATED, USER_DATA_FETCHED, ELIGIBILITY_PASSED, BANK_VERIFIED, ANNUITY_SELECTED, DOCS_UPLOADED, CONSENT_PENDING, CRA_SUBMITTED, COMPLETED_SUCCESS, COMPLETED_FAILURE, CANCELLED)
- `corpusDetails` (JSON/Object)
- `eligibilityResult` (JSON/Object)
- `bankVerificationDetails` (JSON/Object)
- `selectedAnnuity` (JSON/Object)
- `documentBundleReference` (String - e.g., ID or path to bundle)
- `craSubmissionDetails` (JSON/Object - includes CRA Ack ID)
- `finalAcknowledgementId` (String)
- `createdAt` (Timestamp)
- `updatedAt` (Timestamp)
- `failureDetails` (JSON/Object - if status is failure)

_(Other entities like User, Nominee, BankAccount, DocumentMetadata, AnnuityScheme reside in their respective microservices.)_

### 5.2. Data Integrity and Security

- **Data at Rest:** Encrypt sensitive data in databases (e.g., PII, financial details) using mechanisms like TDE or application-level encryption.
- **Data in Transit:** All inter-service communication (gRPC, REST, Kafka, Redis) must use TLS/SSL.
- **Input Validation:** Rigorous validation at API gateways, facade services, and within Temporal activities.
- **Checksums for Documents:** As described in [Section 3.1.8](#318-document-management-service), use checksums (e.g., SHA-256) to ensure file integrity.
- **Audit Trails:** Critical actions and state changes within the workflow and by services are logged to an immutable audit log (potentially via Kafka to a dedicated audit service).
- **Access Control:** Role-Based Access Control (RBAC) for APIs and system operations.
- **PII Handling:** Minimize PII exposure in logs and non-essential API responses. Follow data masking/anonymization practices where applicable.
- **Secrets Management:** Use a secure vault (e.g., HashiCorp Vault, AWS KMS) for managing API keys, database credentials, and other secrets.

---

## 6. Appendices

### 6.1. Eligibility Evaluation Logic (Illustrative)

**(This is a placeholder. Actual rules must come from PFRDA guidelines and business SMEs.)**

```
function evaluateEligibility(pran, category, corpus, dob, doj_nps, other_params):
  // SA (Superannuation)
  if category == "Superannuation":
    age = calculateAge(dob)
    if age < PFRDA_SUPERANNUATION_AGE:
      return {eligible: false, reason: "Below superannuation age"}
    if corpus <= 500000:
      return {eligible: true, lumpsum_percent: 100, annuity_percent: 0}
    else:
      return {eligible: true, lumpsum_percent: 60, annuity_percent: 40} // Max lumpsum

  // PE (Premature Exit)
  if category == "PrematureExit":
    years_of_service = calculateServiceYears(doj_nps)
    if years_of_service < PFRDA_PE_MIN_SERVICE_YEARS:
      return {eligible: false, reason: "Insufficient service years for PE"}
    // Add more specific PE reason checks if needed
    if corpus <= 250000:
      return {eligible: true, lumpsum_percent: 100, annuity_percent: 0}
    else:
      return {eligible: true, lumpsum_percent: 20, annuity_percent: 80} // Max lumpsum

  // IC (Incapacitation)
  if category == "Incapacitation":
    if !other_params.hasValidMedicalCertificate: // This check would happen before calling rule engine
        return {eligible: false, reason: "Valid medical certificate for incapacitation required"}
    // PFRDA rules might allow full withdrawal irrespective of corpus or service years
    // This is an assumption and needs verification.
    return {eligible: true, lumpsum_percent: 100, annuity_percent: 0}

  return {eligible: false, reason: "Unknown withdrawal category or unhandled case"}
```

### 6.2. Diagram Stubs

This LLD should include the following diagrams embedded or linked:

1.  **High-Level System Architecture Diagram (Section 2.2):**
    - Shows major components: API Gateway, UI/PoP, Withdrawal Orchestration (Temporal), Kafka, Redis, and key domain microservices (User, Document, CRA Gateway, etc.).
    - Illustrates primary request flows and event flows.
2.  **Temporal Workflow Interaction Diagram (Conceptual):**
    - Shows a sample withdrawal workflow (e.g., Superannuation).
    - Depicts the sequence of activities, decision points, signals, and interactions with external services.
3.  **Component Interaction Diagram (Detailed - for a specific scenario):**
    - Example: Sequence diagram for "Bank Account Verification" step, showing UI -> API Gateway -> Temporal Workflow -> VerifyBankAccountActivity -> Payment Gateway Service -> Bank.
4.  **Data Model / ER Diagram (Conceptual for key shared entities or workflow state):**
    - Illustrates the main data entities relevant to the withdrawal process and their relationships.
5.  **Deployment Diagram:**
    - Shows how services (including Temporal cluster, Kafka, Redis) are deployed across infrastructure (e.g., Kubernetes clusters, VMs).

### 6.3. Glossary

- **API:** Application Programming Interface
- **ASP:** Annuity Service Provider
- **CRA:** Central Recordkeeping Agency
- **DDD:** Domain-Driven Design
- **gRPC:** Google Remote Procedure Call (a high-performance RPC framework)
- **HLD:** High-Level Design
- **IC:** Incapacitation (withdrawal category)
- **JSON:** JavaScript Object Notation
- **Kafka (Apache Kafka):** Distributed event streaming platform
- **LLD:** Low-Level Design
- **NPS:** National Pension System
- **OTP:** One-Time Password
- **PE:** Premature Exit (withdrawal category)
- **PFRDA:** Pension Fund Regulatory and Development Authority
- **PII:** Personally Identifiable Information
- **PoP:** Point of Presence
- **PRAN:** Permanent Retirement Account Number
- **Protobuf (Protocol Buffers):** Language-neutral, platform-neutral, extensible mechanism for serializing structured data.
- **RBAC:** Role-Based Access Control
- **Redis:** In-memory data structure store, used as a cache and message broker.
- **REST:** Representational State Transfer (architectural style for APIs)
- **SA:** Superannuation (withdrawal category)
- **Saga Pattern:** A way to manage data consistency across microservices in distributed transactions using a sequence of local transactions.
- **SDK:** Software Development Kit
- **SPF/DKIM/DMARC:** Email authentication methods (Sender Policy Framework, DomainKeys Identified Mail, Domain-based Message Authentication, Reporting & Conformance)
- **Temporal.io:** Open-source, durable execution system for writing and running workflows and activities.
- **TLS/SSL:** Transport Layer Security / Secure Sockets Layer (cryptographic protocols for secure communication)
- **TTL:** Time To Live (for cached data)
- **UUID:** Universally Unique Identifier

---

## 7. Notes on Usage

### 7.1. Customization

This LLD template can be tailored:

- **Microservices (User, Document, etc.):** Each will have its own detailed LLD focusing on internal logic, API contracts specific to that domain, data persistence, and its own scaling/resilience strategies.
- **Temporal Workflows & Activities:** Detailed design for each specific workflow (SA, PE, IC) and its activities, including retry policies, compensation logic, and input/output schemas, should be documented further.
- **Infrastructure Components:** LLDs for managing Kafka, Redis, or Temporal clusters would cover operational aspects, monitoring, and capacity planning.

### 7.2. Focus of the LLD

This LLD is implementation-focused for the **NPS Withdrawal Orchestration System**. It aims to:

- Detail internal structures, algorithms, and service/workflow flows.
- Capture interface-level logic between orchestrated components and dependent microservices.
- Support developers in writing modular, testable, resilient, and performant code for the withdrawal process.

It deliberately excludes (or only touches upon):

- Detailed LLDs of individual dependent microservices (they should have their own).
- Business requirements (covered in BRD/FRD).
- UI/UX design (covered in UI specifications).
- Detailed operational runbooks (covered in Ops documentation).
