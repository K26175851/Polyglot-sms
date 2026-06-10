# Polyglot Distributed SMS Service

A microservices-based SMS notification system split into two services:

| Service | Language / Framework | Port |
|---------|---------------------|------|
| **SMS Sender** | Java 17 / Spring Boot 3 | 8080 |
| **SMS Store** | GoLang 1.21 / net/http | 8081 |

---

## How to Run Locally

You need **4 terminals** open at the same time.

### Step 1 — Open Docker Desktop
Open Docker Desktop and wait until the whale icon stops animating and shows **"Engine running"**.

### Step 2 — Start Infrastructure (Terminal 1)
Open Git Bash, go to the project folder and run:

```bash
cd ~/Downloads/polyglot-sms/polyglot-sms
docker compose up -d zookeeper kafka redis mongodb
```

Check all 4 are running:

```bash
docker compose ps
```

Wait until all show **healthy** (about 30 seconds).

### Step 3 — Start SMS Sender Java Service (Terminal 2)
```bash
cd ~/Downloads/polyglot-sms/polyglot-sms/sms-sender
mvn spring-boot:run
```

Wait until you see:
```
Started SmsSenderApplication in X seconds
```

### Step 4 — Start SMS Store Go Service (Terminal 3)
```bash
cd ~/Downloads/polyglot-sms/polyglot-sms/sms-store
export KAFKA_BROKERS=localhost:9092
go run ./cmd/server/...
```

Wait until you see:
```
[Main] Connected to MongoDB uri=mongodb://localhost:27017
[Main] HTTP server listening on :8081
[KafkaConsumer] Starting consumer topic=sms-events
```

Keep this terminal visible — this is where the GoLang service logs appear in real time when events are received from Kafka.

### Step 5 — Run the End-to-End Demo (Terminal 4)
Open Git Bash and run:

```bash
cd ~/Downloads/polyglot-sms/polyglot-sms
./demo.sh
```

---

## Endpoints

### SMS Sender — `http://localhost:8080`

---

#### POST `/v1/sms/send`
Sends an SMS. Checks the Redis block list first, calls the mocked 3P vendor, then publishes the event to Kafka for the GoLang service to store.

**Request Body:**
```json
{
  "userId":      "user123",
  "phoneNumber": "+911234567890",
  "message":     "Hello World"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `userId` | string | yes | Unique identifier of the user |
| `phoneNumber` | string | yes | Phone number in international format e.g. `+911234567890` |
| `message` | string | yes | SMS message body |

**Responses:**

| HTTP Status | Status Field | Meaning |
|-------------|-------------|---------|
| 200 | `SUCCESS` | SMS sent successfully, event published to Kafka |
| 403 | `BLOCKED` | User is on the Redis block list, SMS not sent |
| 502 | `FAIL` | 3P vendor returned a failure |
| 400 | — | Validation failed, missing or invalid fields |

**Example success response:**
```json
{
  "status": "SUCCESS",
  "message": "Message queued for delivery",
  "userId": "user123",
  "phoneNumber": "+911234567890"
}
```

**Example blocked response:**
```json
{
  "status": "BLOCKED",
  "message": "User is blocked and cannot receive SMS",
  "userId": "user123",
  "phoneNumber": "+911234567890"
}
```

---

#### POST `/v1/sms/block/{userId}`
Adds a user to the Redis block list. Any SMS send attempt for this user will be rejected with HTTP 403.

**Path Parameter:** `userId` — the user to block

**Example:**
```
POST http://localhost:8080/v1/sms/block/user123
```

**Response:**
```
User user123 has been blocked.
```

---

#### DELETE `/v1/sms/block/{userId}`
Removes a user from the Redis block list.

**Path Parameter:** `userId` — the user to unblock

**Example:**
```
DELETE http://localhost:8080/v1/sms/block/user123
```

**Response:**
```
User user123 has been unblocked.
```

---

#### GET `/actuator/health`
Health check for the Java SMS Sender service.

**Response:**
```json
{
  "status": "UP"
}
```

---

### SMS Store — `http://localhost:8081`

---

#### GET `/v1/user/{userId}/messages`
Retrieves all SMS records stored for a given user, sorted newest first. Records are written to MongoDB by the GoLang Kafka consumer when it receives events from the Java service.

**Path Parameter:** `userId` — the user whose message history to retrieve

**Example:**
```
GET http://localhost:8081/v1/user/user123/messages
```

**Response:**
```json
[
  {
    "id":             "6a2906ece94a8901423d4f2c",
    "userId":         "user123",
    "phoneNumber":    "+911234567890",
    "message":        "Second message!",
    "status":         "SUCCESS",
    "vendorResponse": "Message queued for delivery",
    "sentAt":         "2026-06-10T06:40:43.994Z",
    "storedAt":       "2026-06-10T06:40:44.152Z"
  },
  {
    "id":             "6a2906ebe94a8901423d4f2b",
    "userId":         "user123",
    "phoneNumber":    "+911234567890",
    "message":        "Hello from demo!",
    "status":         "SUCCESS",
    "vendorResponse": "Message queued for delivery",
    "sentAt":         "2026-06-10T06:40:42.572Z",
    "storedAt":       "2026-06-10T06:40:43.369Z"
  }
]
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | MongoDB document ID |
| `userId` | string | User who the SMS was sent to |
| `phoneNumber` | string | Destination phone number |
| `message` | string | SMS message body |
| `status` | string | `SUCCESS` or `FAIL` |
| `vendorResponse` | string | Message returned by the 3P vendor |
| `sentAt` | string | Timestamp when Java service sent the SMS (ISO 8601) |
| `storedAt` | string | Timestamp when GoLang service stored the record (ISO 8601) |

Returns an empty array `[]` if no records exist for the userId.

---

#### GET `/health`
Health check for the GoLang SMS Store service.

**Response:**
```json
{
  "status": "UP"
}
```

---

## Running Unit Tests

### SMS Sender (Java)
```bash
cd sms-sender
mvn test
```

Expected output:
```
[INFO] Tests run: 10, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

### SMS Store (Go)
```bash
cd sms-store
go test ./...
```

Expected output:
```
ok  github.com/sms/sms-store/internal/service
ok  github.com/sms/sms-store/internal/handler
```

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
│       │   │   ├── AppConfig.java
│       │   │   ├── KafkaConfig.java
│       │   │   └── RedisConfig.java
│       │   ├── controller/
│       │   │   ├── SmsController.java
│       │   │   └── GlobalExceptionHandler.java
│       │   ├── kafka/
│       │   │   └── SmsEventProducer.java
│       │   ├── model/
│       │   │   ├── SmsRequest.java
│       │   │   ├── SmsResponse.java
│       │   │   ├── SmsEvent.java
│       │   │   └── VendorResponse.java
│       │   └── service/
│       │       ├── SmsSenderService.java
│       │       ├── BlockedUserService.java
│       │       └── SmsVendorService.java
│       └── test/java/com/sms/sender/service/
│           ├── SmsSenderServiceTest.java
│           └── BlockedUserServiceTest.java
│
└── sms-store/                      # GoLang service (port 8081)
    ├── Dockerfile
    ├── go.mod
    ├── config/
    │   └── config.go
    ├── cmd/server/
    │   └── main.go
    └── internal/
        ├── model/
        │   └── sms_record.go
        ├── repository/
        │   └── sms_repository.go
        ├── service/
        │   ├── sms_service.go
        │   └── sms_service_test.go
        ├── handler/
        │   ├── sms_handler.go
        │   └── sms_handler_test.go
        └── kafka/
            └── consumer.go
```
