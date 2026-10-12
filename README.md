# UPI Offline Mesh

> **A learning-focused prototype that simulates UPI-style payments over an offline, Bluetooth-inspired mesh network.**  
> Built with Java, Spring Boot, Spring Data JPA, Thymeleaf, and an H2 in-memory database.

When internet connectivity is unavailable, a payment packet can be passed between nearby devices and uploaded by a bridge device when connectivity returns. This project simulates that flow locally, so you can explore encrypted payloads, packet forwarding, duplicate handling, and transaction settlement without physical Bluetooth hardware or a payment-network integration.

> [!WARNING]
> **This is a demonstration project, not a real UPI application.** It does not connect to NPCI, banks, UPI rails, or real payment accounts. Do not enter real PINs or financial information, and do not use it to move real money.

## Highlights

- **Hybrid encryption:** RSA-OAEP with SHA-256 protects a per-packet AES-256 key; AES-GCM encrypts and authenticates the payment payload.
- **Offline mesh simulation:** Virtual devices exchange packets through simulated gossip rounds.
- **Bridge ingestion:** A simulated internet-connected device uploads packets to the backend.
- **Idempotency:** A concurrent-safe in-memory claim prevents the same ciphertext from being settled more than once during the cache's retention window.
- **Replay/freshness check:** Old packets are rejected based on a configurable age window.
- **Transactional settlement:** Sender debit, receiver credit, and ledger entry are performed in a database transaction.
- **Concurrency protection:** Account versioning provides optimistic locking as an additional safeguard.
- **Interactive dashboard:** A Thymeleaf dashboard lets you run the flow from a browser.
- **Automated test:** A concurrency test exercises multiple simultaneous deliveries of the same packet.

## Tech Stack

| Area | Technology |
|---|---|
| Language | Java 17 |
| Backend | Spring Boot 3.3.5, Spring Web |
| Persistence | Spring Data JPA, Hibernate |
| Demo database | H2 in-memory |
| UI | Thymeleaf, HTML, CSS, JavaScript |
| Security primitives | RSA-OAEP, AES-256-GCM, SHA-256 |
| Build | Maven Wrapper |
| Testing | JUnit / Spring Boot Test |

## Architecture

```text
Sender simulation
      |
      | Payment instruction
      v
Hybrid encryption
(RSA-OAEP + AES-256-GCM)
      |
      v
Mesh packet (ciphertext + TTL)
      |
      v
Virtual offline devices
      |       |       |
      +--- gossip rounds ---+
                            |
                            v
                  Internet-connected bridge
                            |
                            | POST /api/bridge/ingest
                            v
                    Backend ingestion pipeline
                            |
                +-----------+-----------+
                |                       |
          SHA-256 hash            Decrypt + validate
          idempotency claim       packet freshness
                |                       |
                +-----------+-----------+
                            |
                            v
                  Transactional settlement
                 debit + credit + ledger row
                            |
                            v
                    H2 database + dashboard
```

### Payment lifecycle

1. The demo creates a payment instruction with sender, receiver, amount, a PIN hash, nonce, and timestamp.
2. The instruction is encrypted using a hybrid RSA/AES scheme and wrapped in a mesh packet with a time-to-live (TTL).
3. Virtual devices exchange packets during gossip rounds.
4. A bridge device uploads packets to the backend.
5. The backend hashes the ciphertext and claims it in the idempotency cache.
6. If the packet is new, the backend decrypts it and checks its timestamp.
7. The settlement service updates the sender and receiver balances and records the transaction atomically.

## Getting Started

### Prerequisites

- JDK 17 or newer
- A terminal (PowerShell, Command Prompt, macOS Terminal, or Linux shell)
- Internet access for the first build so Maven can download dependencies

No separate database installation is required. The application uses an H2 in-memory database.

### 1. Clone the repository

```bash
git clone https://github.com/123sumedh/upi-offline-mesh.git
cd upi-offline-mesh
```

If your GitHub repository uses a different URL or name, replace the clone URL and folder name above.

### 2. Run the application

**Windows (PowerShell or Command Prompt):**

```powershell
.\mvnw.cmd spring-boot:run
```

**macOS / Linux:**

```bash
chmod +x mvnw
./mvnw spring-boot:run
```

### 3. Open the dashboard

Visit:

**http://localhost:8080**

The application seeds demo accounts at startup. The database is in memory, so data is reset when the application restarts.

### 4. Run tests

**Windows:**

```powershell
.\mvnw.cmd test
```

**macOS / Linux:**

```bash
./mvnw test
```

To run the concurrency idempotency test by itself:

```bash
./mvnw -Dtest=IdempotencyConcurrencyTest test
```

On Windows, use `.\mvnw.cmd` instead of `./mvnw`.

## Trying the Demo

Use the dashboard at `http://localhost:8080` and follow the controls in order:

1. **Inject a payment** into the mesh.
2. **Run gossip rounds** to distribute the packet between virtual devices.
3. **Flush bridge uploads** to simulate a device regaining internet access and sending packets to the backend.
4. Review the account balances and transaction ledger to see the settlement result.

The default scenario contains four offline phones and one internet-connected bridge. The mesh is simulated in software: it does not use actual Bluetooth or nearby-device discovery.

## API Overview

The application exposes these main endpoints:

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/` | Opens the dashboard |
| `GET` | `/api/server-key` | Returns the server public key and encryption scheme metadata |
| `POST` | `/api/demo/send` | Creates an encrypted demo payment and injects it into a virtual device |
| `GET` | `/api/mesh/state` | Shows virtual-device packet counts and idempotency cache size |
| `POST` | `/api/mesh/gossip` | Runs one simulated gossip round |
| `POST` | `/api/mesh/flush` | Uploads packets held by internet-connected bridge devices |
| `GET` | `/api/accounts` | Returns demo account information |
| `GET` | `/api/transactions` | Returns the transaction ledger |

### Example: create a demo payment

```bash
curl -X POST http://localhost:8080/api/demo/send \
  -H "Content-Type: application/json" \
  -d '{
    "senderVpa": "alice@demo",
    "receiverVpa": "bob@demo",
    "amount": 100,
    "pin": "demo-only-pin",
    "ttl": 5,
    "startDevice": "phone-alice"
  }'
```

Then distribute and upload the packet:

```bash
curl -X POST http://localhost:8080/api/mesh/gossip
curl -X POST http://localhost:8080/api/mesh/flush
```

Repeat the gossip request if you want to explore packet propagation. For exact request and response structures, inspect `ApiController.java`.

## Demo Accounts

The application seeds these sample accounts if the database is empty:

| VPA | Display name | Initial balance |
|---|---|---:|
| `alice@demo` | Alice | ₹5,000.00 |
| `bob@demo` | Bob | ₹1,000.00 |
| `carol@demo` | Carol | ₹2,500.00 |
| `dave@demo` | Dave | ₹500.00 |

These are local demo records only.

## Project Structure

```text
src/
├── main/
│   ├── java/com/demo/upimesh/
│   │   ├── config/       # Application configuration
│   │   ├── controller/   # Dashboard and REST API controllers
│   │   ├── crypto/       # Hybrid encryption and server key handling
│   │   ├── model/        # Entities, repositories, and packet models
│   │   └── service/      # Mesh simulation, ingestion, idempotency, settlement
│   └── resources/
│       ├── templates/    # Thymeleaf dashboard
│       └── application.properties
└── test/
    └── java/com/demo/upimesh/
        └── IdempotencyConcurrencyTest.java
```

## Important Engineering Concepts

### Hybrid encryption

The payment payload is encrypted with AES-GCM, while RSA-OAEP protects the AES session key. AES-GCM also authenticates the encrypted payload, so modifications are detected during decryption.

### Idempotent ingestion

A SHA-256 hash of the ciphertext is used as the idempotency key. An atomic claim allows only one concurrent request to process a particular packet while the key remains in the cache.

### Transactional settlement

The settlement layer performs the sender debit, receiver credit, and ledger write as one database transaction. Account versioning adds optimistic concurrency control.

### Replay/freshness protection

The backend checks whether a packet timestamp falls within the configured age window. This is a basic demonstration of freshness validation, not a complete production anti-fraud protocol.

## Configuration

Key settings are in `src/main/resources/application.properties`:

```properties
server.port=8080
spring.datasource.url=jdbc:h2:mem:upimesh
upi.mesh.idempotency-ttl-seconds=86400
upi.mesh.packet-max-age-seconds=86400
```

- `server.port`: local HTTP port.
- `upi.mesh.idempotency-ttl-seconds`: how long packet hashes are retained by the idempotency service.
- `upi.mesh.packet-max-age-seconds`: maximum permitted packet age.

## Limitations and Production Considerations

This project is intentionally a **local simulation**. Before anything like it could be used for real payments, it would need a substantial security, reliability, and regulatory redesign, including:

- Real device-to-device transport and authenticated peer discovery.
- A trusted, threat-modelled device key lifecycle and secure key storage.
- Strong sender authorization, transaction signing, PIN handling, and fraud controls.
- Durable, distributed idempotency storage shared across backend instances.
- Careful handling of failed/partial bridge uploads and idempotency-claim recovery.
- Production database configuration, observability, audit trails, and disaster recovery.
- A formal security review, privacy review, and applicable payment-system approvals.

The current in-memory idempotency cache is not shared between application instances and is lost on restart. The H2 database is also in-memory. The public-key encryption demonstration should not be interpreted as proof that the full payment protocol is secure.

## Future Improvements

- Replace the simulator with a real BLE transport layer.
- Add durable idempotency keys and explicit packet-processing states.
- Add API authentication, authorization, request validation, and rate limiting.
- Add integration tests for invalid, tampered, expired, and duplicate packets.
- Add metrics and structured audit events.
- Provide a mobile client for composing and forwarding packets.

## Contributing

Contributions and suggestions are welcome.

1. Fork the repository.
2. Create a feature branch.
3. Make focused changes and add tests where appropriate.
4. Open a pull request describing the change and how it was tested.

## License

No license is currently specified. Unless a license is added to the repository, reuse and redistribution permissions may be limited. Add a `LICENSE` file if you intend to publish this project for reuse.

---

**Built as a backend engineering and distributed-systems learning project.**
