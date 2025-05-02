## **Product Requirements Document (PRD)**

### **1. Introduction**

This document outlines the requirements for the **Event-Driven Architecture (EDA)** that will enable asynchronous communication between microservices. The architecture will be powered by **Kafka**, with services leveraging **gRPC** and **REST** APIs for communication. The goal is to provide a scalable, resilient, and loosely coupled system that ensures data integrity, fault tolerance, and efficient real-time processing.

### **2. Goals & Objectives**

- **Asynchronous Communication:** Decouple microservices through event-driven communication using Kafka as the central messaging platform.
- **Scalability & Resilience:** Build a scalable architecture with fault-tolerant mechanisms like retries, backoffs, and dead-letter queues.
- **Real-time Processing:** Ensure services process events in real-time with low latency.
- **Data Consistency:** Ensure that microservices have a consistent view of the data while avoiding tight coupling.
- **Ease of Integration:** Services should integrate seamlessly with other services via Kafka topics.

### **3. Key Components**

1. **Kafka Cluster**: The core event-streaming system facilitating communication between microservices.
2. **Microservices**: Each service will be a producer or consumer of Kafka events.
3. **Event Schema Registry**: Manages and validates event schemas to ensure compatibility.
4. **Message Brokers (Kafka)**: Ensures reliable delivery of events between services.
5. **gRPC/REST APIs**: Allows external and internal communication, triggering events or consuming them.
6. **Infrastructure Layer**: Provides abstraction for Kafka and communication layers, such as consumers and producers.

---

### **4. Functional Requirements**

#### **4.1 Kafka Cluster & Topics**

- **Kafka Cluster**: A centralized Kafka cluster will be used for all services to produce and consume messages.

  - **Kafka Brokers**: 3 Kafka brokers deployed in a highly available configuration.
  - **Kafka Topics**:

    - `order-service.order.created`
    - `payment-service.payment.completed`
    - `inventory-service.stock.updated`

  - Each topic will have its own **consumer group** per service to ensure events are consumed at scale.

#### **4.2 Microservices**

Each microservice has a clear responsibility and will produce or consume specific events:

1. **Order Service** (Producer):

   - Event: `order.created`, `order.updated`
   - **gRPC/REST API** will trigger event publishing when an order is created or updated.
   - **Kafka Producer**: Publishes events to Kafka.

2. **Payment Service** (Consumer):

   - Event: `order.created` (from Order Service)
   - Consumes the `order.created` event, processes it, and triggers `payment.completed` event.
   - **Kafka Consumer**: Subscribes to Kafka topics and processes events.

3. **Inventory Service** (Consumer):

   - Event: `payment.completed` (from Payment Service)
   - Consumes the `payment.completed` event to trigger inventory adjustments.
   - **Kafka Consumer**: Subscribes to Kafka topics and processes events.

#### **4.3 Event Producer & Consumer Logic**

Each microservice will implement a **Producer** and/or **Consumer** module to handle Kafka interactions.

**Producer Module:**

- Will serialize data and send events to Kafka.
- Example (Order Service):

  ```go
  func publishOrderCreatedEvent(order Order) {
      writer := kafka.NewWriter(kafka.WriterConfig{
          Addr: kafka.TCP("kafka-broker:9092"),
          Topic: "order-service.order.created",
      })
      err := writer.WriteMessages(context.Background(), kafka.Message{Value: []byte(order.ID)})
      if err != nil {
          log.Fatalf("Failed to write message: %v", err)
      }
  }
  ```

**Consumer Module:**

- Listens for events and processes them as required.
- Example (Payment Service):

  ```go
  func consumeOrderCreatedEvent() {
      reader := kafka.NewReader(kafka.ReaderConfig{
          Brokers: []string{"kafka-broker:9092"},
          Topic: "order-service.order.created",
          GroupID: "payment-service-consumer",
      })
      for {
          msg, err := reader.ReadMessage(context.Background())
          if err != nil {
              log.Fatalf("Error reading message: %v", err)
          }
          // Process event here
      }
  }
  ```

#### **4.4 Event Schema Registry**

To ensure that the message format is consistent and valid:

- Use **Schema Registry** (e.g., Confluent Schema Registry) to store and manage schemas for events like `order.created`, `payment.completed`.
- Event schemas will be in **Avro** or **JSON** format.
- Ensure backward compatibility when evolving schemas.

#### **4.5 Infrastructure Layer**

The infrastructure layer will abstract Kafka-related operations, including:

- **Kafka Producer/Consumer Manager**: Manages Kafka interactions like subscribing to topics, producing events, and error handling.
- **Schema Validator**: Ensures that messages conform to the defined schema before sending them to Kafka.

#### **4.6 gRPC/REST APIs**

- **REST/gRPC APIs** will trigger events or retrieve data.

  - REST APIs will be used for legacy systems and integration with external clients.
  - gRPC APIs will be used internally for performance reasons.

Example of **Order Service API**:

```go
// gRPC example for triggering an event
func (s *OrderServiceServer) CreateOrder(ctx context.Context, req *CreateOrderRequest) (*CreateOrderResponse, error) {
    // Create order logic
    publishOrderCreatedEvent(order)
    return &CreateOrderResponse{Message: "Order Created"}, nil
}
```

---

### **5. Non-Functional Requirements**

#### **5.1 Scalability & Resilience**

- **Kafka**: Kafka provides horizontal scalability. Add more brokers if necessary.
- **Microservices**: Each microservice can scale independently based on event traffic.

#### **5.2 Fault Tolerance**

- Use **Dead Letter Queue (DLQ)** for messages that fail to process after multiple attempts.
- Implement **retry mechanisms** with exponential backoff.

#### **5.3 Monitoring & Observability**

- Use **Prometheus** and **Grafana** to monitor Kafka broker health, consumer lag, and event processing times.
- Use **Elastic Stack** or **Fluentd** for centralized log aggregation.

#### **5.4 Security**

- Kafka should use **TLS encryption** for data in transit.
- Implement **SASL** for authentication and **ACLs** for authorization in Kafka.
- Microservices should validate inputs to prevent malicious events.

---

### **6. DevOps Tasks**

- **Kafka Deployment**:

  - Deploy Kafka on Kubernetes using **Strimzi** operator.
  - Manage **Kafka clusters** and **topics** for each environment (Dev, Staging, Production).

- **Service Deployment**:

  - Use **Helm** to package and deploy microservices on Kubernetes.
  - Deploy **Schema Registry** for managing Avro schemas.

- **Monitoring Setup**:

  - Set up **Prometheus** for monitoring Kafka metrics and microservices.
  - Set up **Grafana Dashboards** to visualize Kafka metrics and microservices' health.

- **CI/CD Pipeline**:

  - Use **Jenkins** or **GitLab CI/CD** to build, test, and deploy microservices.
  - Automate the deployment of Kafka services using Helm charts.

---

### **7. High-Level Architecture Diagram**

```plaintext
  +----------------+                +----------------+                 +----------------+
  | Order Service  |  ---> [order-created] ---> | Payment Service | ---> [payment-completed] ---> | Inventory Service |
  +----------------+                +----------------+                 +----------------+
         |                                |                                 |
       [REST/gRPC]                   [Kafka]                           [Kafka]
         |                                |                                 |
  +----------------+                +----------------+                 +----------------+
  |    External    |  ---> [API Call] ---> |   Kafka      | ---> [Consume] ---> |  Kafka Consumer  |
  |    Client      |                +----------------+                 +----------------+
  +----------------+
```

---

### **8. Summary**

This **Event-Driven Microservices Architecture** leverages **Kafka** for scalable and reliable event-streaming between services. The architecture ensures that the system is fault-tolerant, scalable, and easily integrable with external and internal clients via **REST/gRPC APIs**. Kafka's **topic-based** communication model ensures loose coupling between services, making it easier to manage and evolve the system over time. The **Schema Registry** ensures consistency across different microservices while avoiding breaking changes.

This architecture is designed to meet both functional and non-functional requirements, providing a robust solution for event-driven communication, with considerations for security, scalability, resilience, and monitoring.

---

### **9. Next Steps**

- Finalize Kafka setup and deploy the cluster.
- Implement core microservices and integrate them with Kafka for event publishing and consumption.
- Set up monitoring and logging solutions.
- Develop CI/CD pipelines for seamless deployments and updates.
