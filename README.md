[README.md](https://github.com/user-attachments/files/26666245/README.md)
# Event-Driven Order Processing System

A production-style distributed microservices system built with Java and Spring Boot,
using Apache Kafka for asynchronous event streaming across independently deployable services.

## Architecture

```
Client Request
     ↓
[Order Service :8080]
     ↓ publishes to Kafka
     ├──────────────────────────────┐
     ↓                              ↓
[Stock Service :8082]      [Payment Service :8083]
 MongoDB · checks inventory  PostgreSQL · processes payment
     ↓                              ↓
 stock-events topic          payment-events topic
     └──────────┬───────────────────┘
                ↓
     [Kafka Streams Join Window]
      correlates both responses
                ↓
     [Order Service] publishes final status
                ↓
     [Notification Service :8084]
      logs outcome + sends alert
```

## Services

| Service | Port | Database | Role |
|---|---|---|---|
| Order | 8080 | PostgreSQL | Orchestrator, Kafka Streams joins |
| Stock | 8082 | MongoDB | Inventory validation |
| Payment | 8083 | PostgreSQL | Payment processing |
| Notification | 8084 | — | Customer alerts |
| Common | — | — | Shared DTOs, enums |

## Tech Stack

- **Language:** Java 11+
- **Framework:** Spring Boot 2.6.x
- **Messaging:** Apache Kafka + Zookeeper
- **Stream Processing:** Kafka Streams (join window)
- **Databases:** PostgreSQL, MongoDB
- **Infrastructure:** Docker, Docker Compose, Kubernetes
- **Testing:** JUnit 5, Mockito, JaCoCo
- **Build:** Maven (multi-module)

## Key Features

- Asynchronous inter-service communication via Kafka topics
- Kafka Streams **join window** — correlates stock + payment responses in real time
- At-least-once delivery with dead-letter queue handling
- Distributed rollback logic — if stock fails after payment succeeds, payment is refunded
- Polyglot persistence — right database for each service
- Kubernetes deployment with liveness/readiness probes and horizontal pod autoscaling
- Full Docker Compose local setup — one command spins everything up

## Running Locally

**Prerequisites:** Docker, Docker Compose, Java 11+, Maven

```bash
# 1. Start all infrastructure
docker-compose up -d

# 2. Build all modules
mvn clean install

# 3. Run each service
mvn -pl order spring-boot:run
mvn -pl stock spring-boot:run
mvn -pl payment spring-boot:run
mvn -pl notification spring-boot:run
```

## Infrastructure (Docker Compose)

| Service | Port | Purpose |
|---|---|---|
| Kafka | 29092 | Message broker |
| Zookeeper | 2181 | Kafka coordination |
| MongoDB | 27017 | Stock service DB |
| Mongo Express | 8081 | MongoDB UI |
| PostgreSQL | 5432 | Order/Payment DB |
| pgAdmin | 16543 | PostgreSQL UI |
| Schema Registry | 8085 | Avro schema management |
| Kafdrop | 9000 | Kafka monitoring UI |

## Event Flow

1. Client sends order → Order Service publishes to `order-events`
2. Stock Service consumes → validates inventory → publishes to `stock-events`
3. Payment Service consumes → processes payment → publishes to `payment-events`
4. Kafka Streams joins both within 5-second window → Order Service publishes to `order-final-events`
5. Notification Service consumes final status → alerts customer

## Failure Scenarios Handled

| Stock | Payment | Action |
|---|---|---|
| ✅ Success | ✅ Success | Order confirmed |
| ❌ Failed | ✅ Success | Rollback payment, notify customer |
| ✅ Success | ❌ Failed | Rollback stock reservation, notify customer |
| ❌ Failed | ❌ Failed | Order rejected |

## Testing

```bash
mvn test                    # run all tests
mvn test jacoco:report      # generate coverage report
```

Coverage enforced via JaCoCo across all modules.

## License

Apache 2.0
