# Polyglot Distributed SMS Service

A microservices-based SMS notification system split into two services:

| Service | Language / Framework | Port |
|---------|---------------------|------|
| **SMS Sender** | Java 17 / Spring Boot 3 | 8080 |
| **SMS Store** | GoLang 1.21 / net/http | 8081 |

---

## Architecture

```
Client
  │
  ▼ POST /v1/sms/send
┌─────────────────────┐        ┌──────────┐
│   SMS Sender (Java) │──────▶│  Redis   │  (blocked-user set)
│   Spring Boot :8080 │        └──────────┘
│                     │
│  1. Check Redis     │        ┌──────────────────┐
│  2. Call 3P Vendor  │──────▶│ 3P SMS Vendor    │  (mocked)
│     (mocked)        │        └──────────────────┘
│  3. Publish to      │
│     Kafka           │──────▶┌──────────┐
└─────────────────────┘        │  Kafka   │  topic: sms-events
                               └────┬─────┘
                                    │ consume
                               ┌────▼─────────────────┐
                               │  SMS Store (Go)       │
                               │  net/http :8081       │
                               │                       │
                               │  4. Decode event      │
                               │  5. Store in MongoDB  │──▶ MongoDB
                               └───────────────────────┘
                                    │
                                    ▼ GET /v1/user/{userId}/messages
                                  Client
```

---

## Data Flow (step-by-step)

1. **Client** → `POST /v1/sms/send` → **SMS Sender**
2. SMS Sender checks the **Redis blocked-user set**; returns `403` if blocked.
3. SMS Sender calls the **(mocked) 3P SMS Vendor** → `SUCCESS` or `FAIL`.
4. SMS Sender publishes an `SmsEvent` to **Kafka** topic `sms-events`.
5. SMS Sender returns the response to the client.
6. **SMS Store** Kafka consumer picks up the event, stores it in **MongoDB**.
7. **Client** → `GET /v1/user/{userId}/messages` → **SMS Store** → MongoDB.

---

## Prerequisites

- **Docker Desktop** — runs Kafka, Redis, MongoDB
- **Java 17+** — [Download](https://adoptium.net/)
- **Maven** — [Download](https://maven.apache.org/download.cgi)
- **Go 1.21+** — [Download](https://go.dev/dl/)
- **Git Bash** — for running `demo.sh` on Windows

---

## How to Run Locally (Step-by-Step)

### Step 1 — Open Docker Desktop
Open Docker Desktop and wait until the whale icon stops animating and shows **"Engine running"**.

### Step 2 — Start Infrastructure
Open Git Bash, navigate to the project folder and run:

```bash
cd ~/Downloads/polyglot-sms/polyglot-sms
docker compose up -d zookeeper kafka redis mongodb
```

Verify all 4 are running:

```bash
docker compose ps
```

All should show **healthy**. Wait about 30 seconds if they don't immediately.

### Step 3 — Start SMS Sender (Java)
Open a **new terminal** in VS Code and run:

```bash
cd sms-sender
mvn spring-boot:run
```

Wait until you see:
```
Started SmsSenderApplication in ... seconds
```

### Step 4 — Start SMS Store (Go)
Open another **new terminal** in git bash and run:

```bash
cd sms-store
export KAFKA_BROKERS=localhost:9092
go run ./cmd/server/...
```

Wait until you see:
```
[Main] Connected to MongoDB
[Main] HTTP server listening on :8081
[KafkaConsumer] Starting consumer topic=sms-events
```

### Step 5 — Run the Demo
Open another **new terminal** (Git Bash) and run:

```bash
cd ~/Downloads/polyglot-sms/polyglot-sms
./demo.sh
```

---

## Demo Script Output (demo.sh)

The `demo.sh` script runs the full end-to-end flow automatically. Here is the expected output:

```
[INFO]  Step 0 – Health checks
{ "status": "UP" }
[INFO]  SMS Store : UP

[INFO]  Step 1 – Send an SMS for user 'user123'
{
  "status": "SUCCESS",
  "message": "Message queued for delivery",
  "userId": "user123",
  "phoneNumber": "+911234567890"
}
[INFO]  Vendor status: SUCCESS

[INFO]  Step 2 – Send a second SMS for user 'user123'
{
  "status": "SUCCESS",
  "message": "Message queued for delivery",
  "userId": "user123",
  "phoneNumber": "+911234567890"
}

[INFO]  Step 3 – Block user 'blockedUser' and attempt to send
User blockedUser has been blocked.
[INFO]  HTTP status for blocked user: 403  (expected 403)

[INFO]  Step 4 – Wait 3 s for Kafka event to reach SMS Store, then retrieve history
[
  {
    "id": "6a19435c7ed1f54ce3c1d938",
    "userId": "user123",
    "phoneNumber": "+911234567890",
    "message": "Second message!",
    "status": "SUCCESS",
    "vendorResponse": "Message queued for delivery",
    "sentAt": "2026-05-29T07:42:20.394Z",
    "storedAt": "2026-05-29T07:42:20.666Z"
  },
  {
    "id": "6a19435b7ed1f54ce3c1d937",
    "userId": "user123",
    "phoneNumber": "+911234567890",
    "message": "Hello from demo!",
    "status": "SUCCESS",
    "vendorResponse": "Message queued for delivery",
    "sentAt": "2026-05-29T07:42:19.56Z",
    "storedAt": "2026-05-29T07:42:19.811Z"
  }
]
[INFO]  Total SMS records stored for user123: 2

[INFO]  Step 5 – Unblock 'blockedUser'
User blockedUser has been unblocked.
[INFO]  blockedUser unblocked successfully.

[INFO]  Demo complete!
```

What each step demonstrates:

| Step | What it proves |
|------|---------------|
| Step 0 | Both services are healthy and running |
| Step 1 | `POST /v1/sms/send` works, 3P vendor mock returns SUCCESS |
| Step 2 | Multiple SMS messages per user are stored correctly |
| Step 3 | Redis block list works — blocked user gets HTTP 403, SMS not sent |
| Step 4 | Full Kafka → Go consumer → MongoDB pipeline working; history API returns records sorted newest first |
| Step 5 | Redis unblock works correctly |

> **Note:** If you run `demo.sh` multiple times, Step 4 will show more than 2 records — this is expected behaviour since all previous runs are persisted in MongoDB.

---

## API Reference

### SMS Sender — `http://localhost:8080`

#### `POST /v1/sms/send`
Send an SMS message.

**Request body:**
```json
{
  "userId":      "user123",
  "phoneNumber": "+911234567890",
  "message":     "Hello World"
}
```

**Responses:**

| HTTP | Status field | Meaning |
|------|-------------|---------|
| 200  | `SUCCESS`   | SMS sent and event published to Kafka |
| 403  | `BLOCKED`   | userId is on the Redis block list |
| 502  | `FAIL`      | 3P vendor returned failure |
| 400  | —           | Validation error (missing or invalid fields) |

---

#### `POST /v1/sms/block/{userId}`
Add a user to the Redis block list.

---

#### `DELETE /v1/sms/block/{userId}`
Remove a user from the Redis block list.

---

### SMS Store — `http://localhost:8081`

#### `GET /v1/user/{userId}/messages`
Retrieve all SMS records for a user, sorted newest first.

**Response:**
```json
[
  {
    "id":             "6a19435c7ed1f54ce3c1d938",
    "userId":         "user123",
    "phoneNumber":    "+911234567890",
    "message":        "Hello!",
    "status":         "SUCCESS",
    "vendorResponse": "Message queued for delivery",
    "sentAt":         "2026-05-29T07:42:20.394Z",
    "storedAt":       "2026-05-29T07:42:20.666Z"
  }
]
```

---

#### `GET /health`
Health check — returns `{"status":"UP"}`.

---

## Running Unit Tests

### SMS Sender (Java)

Open a terminal in the `sms-sender` folder and run:

```bash
cd sms-sender
mvn test
```

Expected output:
```
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

Tests included:

| Test File | Tests | What it covers |
|-----------|-------|----------------|
| `SmsSenderServiceTest` | 5 | Happy path, blocked user, vendor fail, vendor exception, Kafka failure isolation |
| `BlockedUserServiceTest` | 5 | Redis block/unblock, fail-open when Redis is down |

---

### SMS Store (Go)

Open a terminal in the `sms-store` folder and run:

```bash
cd sms-store
go test ./...
```

Expected output:
```
ok  github.com/sms/sms-store/internal/service
ok  github.com/sms/sms-store/internal/handler
```

Tests included:

| Test File | Tests | What it covers |
|-----------|-------|----------------|
| `sms_service_test.go` | 5 | StoreEvent success/validation/repo error, GetMessagesByUserID success/error |
| `sms_handler_test.go` | 5 | HTTP 200, empty user returns `[]`, wrong method (405), bad path (404), health check |

---

## Environment Variables (SMS Store)

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_PORT` | `8081` | HTTP server port |
| `MONGO_URI` | `mongodb://localhost:27017` | MongoDB connection URI |
| `KAFKA_BROKERS` | `localhost:9092` | Kafka bootstrap servers |
| `KAFKA_TOPIC` | `sms-events` | Kafka topic to consume |
| `KAFKA_GROUP_ID` | `sms-store-group` | Kafka consumer group ID |

---

## Project Structure

```
polyglot-sms/
├── docker-compose.yml              # Kafka, Zookeeper, Redis, MongoDB
├── demo.sh                         # End-to-end demonstration script
├── README.md
│
├── sms-sender/                     # Java Spring Boot service (port 8080)
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/sms/sender/
│       │   ├── SmsSenderApplication.java
│       │   ├── config/
│       │   │   ├── AppConfig.java          # RestTemplate bean
│       │   │   ├── KafkaConfig.java        # Kafka producer + topic creation
│       │   │   └── RedisConfig.java        # RedisTemplate bean
│       │   ├── controller/
│       │   │   ├── SmsController.java      # POST /v1/sms/send + block endpoints
│       │   │   └── GlobalExceptionHandler.java
│       │   ├── kafka/
│       │   │   └── SmsEventProducer.java   # Publishes SmsEvent to Kafka
│       │   ├── model/
│       │   │   ├── SmsRequest.java
│       │   │   ├── SmsResponse.java
│       │   │   ├── SmsEvent.java           # Kafka payload
│       │   │   └── VendorResponse.java
│       │   └── service/
│       │       ├── SmsSenderService.java   # Orchestration logic
│       │       ├── BlockedUserService.java # Redis block-list management
│       │       └── SmsVendorService.java   # Mocked 3P vendor
│       └── test/java/com/sms/sender/service/
│           ├── SmsSenderServiceTest.java
│           └── BlockedUserServiceTest.java
│
└── sms-store/                      # GoLang service (port 8081)
    ├── Dockerfile
    ├── go.mod
    ├── config/
    │   └── config.go               # Env-based configuration
    ├── cmd/server/
    │   └── main.go                 # Entry point, wiring, graceful shutdown
    └── internal/
        ├── model/
        │   └── sms_record.go       # SmsRecord struct + FlexTime for Java Instant
        ├── repository/
        │   └── sms_repository.go   # MongoDB Save + FindByUserID + indexes
        ├── service/
        │   ├── sms_service.go      # Business logic + SmsRepository interface
        │   └── sms_service_test.go
        ├── handler/
        │   ├── sms_handler.go      # net/http handlers, no third-party router
        │   └── sms_handler_test.go
        └── kafka/
            └── consumer.go         # Kafka consumer, commit-after-success
```

---

## Key Design Decisions

- **3P Vendor is mocked** — returns `SUCCESS` (~80%) or `FAIL` (~20%) randomly via `SmsVendorService`. Set `sms.vendor.mock=false` in `application.yml` to wire a real vendor.
- **Fail-open on Redis** — if Redis is unavailable, the block-list check is skipped and the SMS is allowed through (logged as a warning).
- **Kafka publish does not block the response** — if Kafka is down after the vendor call, the error is logged but the API still returns the vendor status to the caller.
- **Kafka consumer commits offset only after successful MongoDB write** — prevents data loss if the Go service crashes mid-processing.
- **FlexTime in Go model** — handles Java's `Instant` which serialises as an epoch number (`1748505600.123`) instead of a JSON string, ensuring correct time parsing.
- **No third-party HTTP router in Go** — pure `net/http` `ServeMux` with manual path parsing as required by the assignment.
- **MongoDB** is used instead of MySQL as specified in the assignment note.
