<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/30eb4314-ca00-4ff1-9b14-9df0bbb1bca3" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f58517ad-f36b-4630-b4c9-42db9d42c73d" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/aa12a8ed-2d63-4cb3-856d-1e0c80111d84" />


# 📡 UPI Offline Mesh

> **A secure offline payment settlement simulation where transactions travel through a Bluetooth-style mesh network and settle when connectivity becomes available.**

![Java](https://img.shields.io/badge/Java-17%2B-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-brightgreen)
![Security](https://img.shields.io/badge/Security-AES--256--GCM%20%2B%20RSA--OAEP-blue)
![Database](https://img.shields.io/badge/Database-H2-lightgrey)
![Architecture](https://img.shields.io/badge/Architecture-Distributed%20Systems-purple)

---

## 🚀 The Idea

Imagine you're in a place with **zero internet connectivity**.

You want to send ₹500 to someone.

Normally, a digital payment cannot be processed because neither device can reach the payment backend.

This project explores a different idea:

1. The sender creates a payment instruction.
2. The payment is encrypted.
3. The encrypted packet is passed between nearby devices.
4. Devices relay the packet through a **Bluetooth-style gossip mesh**.
5. Eventually, a device regains internet connectivity.
6. That device acts as a **bridge node** and uploads the packet to the backend.
7. The backend securely validates and settles the payment.

The important part is this:

> **The devices relaying the payment do not need to be trusted.**

They only transport encrypted data.

---

# ⚠️ Important Disclaimer

This project is a **technical simulation and educational prototype**.

It is **not an implementation of the official NPCI UPI infrastructure** and does not perform real bank transfers.

A more technically accurate description of the system is:

> **Mesh-routed deferred payment settlement**

The project explores the engineering problems involved in transporting payment instructions when internet connectivity is temporarily unavailable.

---

# 🧠 What Makes This Project Interesting?

This is not simply:

```text
Frontend → Backend → Database
```

The project explores several real distributed-systems and fintech engineering problems:

* 🔐 How can an untrusted device carry a payment without reading it?
* 🔁 What happens when the same payment reaches the backend multiple times?
* 🛡️ How do we prevent replay attacks?
* ⚡ What happens when multiple bridge nodes submit the same payment simultaneously?
* 💰 How do we ensure a sender is never debited twice?
* 🔄 How can transactions propagate through a decentralized network?
* 🧵 How do concurrency and database transactions interact?

---

# ✨ Key Features

## 🔐 Hybrid Encryption

Payment instructions are protected using:

* **RSA-OAEP**
* **AES-256-GCM**

The system uses hybrid encryption because RSA is inefficient and size-limited for larger payloads.

The flow is:

```text
Payment Instruction
        │
        ▼
Generate random AES-256 key
        │
        ▼
Encrypt payment payload using AES-GCM
        │
        ▼
Encrypt AES key using RSA-OAEP
        │
        ▼
Create encrypted Mesh Packet
```

The packet contains encrypted data that can safely travel through untrusted devices.

---

## 📡 Bluetooth-Style Mesh Simulation

The project simulates a network of virtual devices.

Each device can:

* Receive packets
* Store packets
* Relay packets
* Participate in gossip rounds

Example:

```text
Alice's Phone
      │
      ▼
Nearby Device
      │
      ▼
Another Device
      │
      ▼
Bridge Device
      │
      ▼
Internet Available
      │
      ▼
Spring Boot Backend
```

In the current implementation, the mesh is simulated in software so the entire system can be demonstrated on a single machine.

---

## 🔁 Gossip-Based Packet Propagation

Packets spread through the simulated network using a gossip-style mechanism.

A device holding a packet shares it with nearby devices.

This naturally creates duplicate copies.

For example:

```text
        ┌────────────┐
        │   Alice    │
        └─────┬──────┘
              │
      ┌───────┴───────┐
      ▼               ▼
   Device A         Device B
      │               │
      ▼               ▼
   Bridge 1         Bridge 2
```

Both bridges may eventually upload the **same payment**.

That creates one of the most important problems in payment systems:

# 💣 Duplicate Processing

---

# 🔥 The Core Engineering Challenge: Idempotency

Imagine Alice sends ₹500.

Because the packet travels through a mesh network:

```text
Bridge A ─────┐
Bridge B ─────┼────► Backend
Bridge C ─────┘
```

All three bridges may submit the exact same payment.

Without protection:

```text
Alice: -₹500
Alice: -₹500
Alice: -₹500
```

💀 Alice gets charged ₹1500.

This project prevents that.

---

## 🧩 Idempotency Strategy

When the backend receives a packet:

```text
Receive Packet
      │
      ▼
SHA-256 Hash Ciphertext
      │
      ▼
Try Atomic Claim
      │
 ┌────┴────┐
 ▼         ▼
NEW      DUPLICATE
 │          │
 ▼          ▼
PROCESS   DROP
```

The system uses an atomic operation similar to:

```java
putIfAbsent()
```

Only one request can successfully claim the payment.

Every duplicate is rejected before settlement.

---

### Why Hash the Ciphertext?

The ciphertext hash is used as the idempotency key.

This is useful because:

### ❌ Not `packetId`

A malicious relay could modify outer metadata.

### ❌ Not plaintext

The backend would need to decrypt before checking duplicates.

### ✅ Ciphertext hash

The backend can:

```text
Hash → Check duplicate → Process only if new
```

This avoids unnecessary cryptographic work for duplicate packets.

---

# 🛡️ Replay Attack Protection

Suppose an attacker captures an old encrypted payment packet.

They could try:

```text
Original Payment → Capture
                      │
                      ▼
               Replay Later
```

The project protects against this using two mechanisms.

---

## 1️⃣ Timestamp Validation

Each payment contains a timestamp.

The backend validates whether the payment is still within the allowed freshness window.

```text
Payment Created
      │
      ▼
Backend receives packet
      │
      ▼
Is timestamp valid?
      │
   ┌──┴──┐
   ▼     ▼
 YES     NO
   │      │
PROCESS  REJECT
```

---

## 2️⃣ Unique Nonce

Every payment contains a unique nonce.

Therefore:

```text
Alice → Bob ₹100
```

and later:

```text
Alice → Bob ₹100
```

are still treated as separate legitimate transactions.

Because:

```text
Payment 1 → Nonce A
Payment 2 → Nonce B
```

A replay of the same packet, however, produces the same encrypted payload and is caught by the idempotency mechanism.

---

# 🔐 Why AES-GCM?

AES-GCM provides:

* 🔒 Confidentiality
* 🛡️ Integrity
* 🔍 Authentication

If someone modifies even a single bit of the encrypted packet:

```text
Original Ciphertext
        │
        ▼
Attacker changes 1 bit
        │
        ▼
AES-GCM Authentication Fails
        │
        ▼
❌ Packet Rejected
```

The backend does not trust the packet simply because it arrived from a bridge device.

The cryptographic authentication must succeed.

---

# 🏗️ System Architecture

```text
┌──────────────────────────────────────────────────────────────┐
│                     SENDER DEVICE (OFFLINE)                  │
│                                                              │
│ PaymentInstruction                                           │
│ {                                                            │
│   sender,                                                   │
│   receiver,                                                 │
│   amount,                                                   │
│   nonce,                                                    │
│   timestamp                                                  │
│ }                                                            │
│                                                              │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                ▼
                    🔐 Hybrid Encryption
                                │
                                ▼
┌──────────────────────────────────────────────────────────────┐
│                         MESH PACKET                          │
│                                                              │
│ packetId                                                     │
│ ttl                                                          │
│ createdAt                                                    │
│ ciphertext 🔒                                                │
│                                                              │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                ▼
                      📡 GOSSIP NETWORK
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
         Device A          Device B          Device C
              │                 │                 │
              └─────────────────┼─────────────────┘
                                │
                                ▼
                        🌉 BRIDGE NODE
                                │
                         Internet Available
                                │
                                ▼
┌──────────────────────────────────────────────────────────────┐
│                     SPRING BOOT BACKEND                      │
│                                                              │
│ /api/bridge/ingest                                           │
│          │                                                   │
│          ▼                                                   │
│ SHA-256(ciphertext)                                          │
│          │                                                   │
│          ▼                                                   │
│ 🔁 Idempotency Check                                         │
│          │                                                   │
│          ▼                                                   │
│ 🔐 Hybrid Decryption                                         │
│          │                                                   │
│          ▼                                                   │
│ 🕒 Freshness Validation                                      │
│          │                                                   │
│          ▼                                                   │
│ 💰 Transaction Settlement                                    │
│          │                                                   │
│          ▼                                                   │
│ 🗄️ Transaction Ledger                                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

# ⚙️ Settlement Pipeline

Every incoming payment follows this pipeline:

```text
1. Receive MeshPacket
          │
          ▼
2. Hash Ciphertext (SHA-256)
          │
          ▼
3. Claim Idempotency Key
          │
      ┌───┴────┐
      ▼        ▼
    NEW     DUPLICATE
      │        │
      ▼        ▼
 Continue    DROP
      │
      ▼
4. Decrypt Packet
      │
      ▼
5. Validate Timestamp
      │
      ▼
6. Validate Payment
      │
      ▼
7. Start Database Transaction
      │
      ├── Debit Sender
      ├── Credit Receiver
      └── Write Ledger Entry
      │
      ▼
8. Commit Transaction
```

---

# 🧵 Concurrency & Exactly-Once Settlement

One of the most important properties of the project is:

> **A payment packet should settle at most once even when delivered concurrently.**

The project includes a concurrency test that simulates multiple bridge nodes delivering the same packet simultaneously.

Conceptually:

```text
Thread 1 ──┐
Thread 2 ──┼────► BridgeIngestionService
Thread 3 ──┘
```

Expected result:

```text
Thread 1 → SETTLED
Thread 2 → DUPLICATE_DROPPED
Thread 3 → DUPLICATE_DROPPED
```

Final balance:

```text
Sender: -₹500 exactly once
Receiver: +₹500 exactly once
```

---

# 💾 Transactional Settlement

Settlement happens inside a database transaction.

Conceptually:

```java
@Transactional
public void settle(...) {

    debitSender();

    creditReceiver();

    writeTransactionLedger();
}
```

This ensures that the financial operations behave as a single unit.

Either:

```text
Debit + Credit + Ledger Entry
```

all succeed,

or the transaction rolls back.

---

# 🔒 Optimistic Locking

Accounts use optimistic locking as an additional layer of protection.

This helps detect conflicting concurrent modifications.

Example:

```text
Transaction A reads balance
Transaction B reads balance

Transaction A updates ✓

Transaction B tries to update
        │
        ▼
Version conflict
        │
        ▼
❌ Retry / Reject
```

This provides defense-in-depth alongside the idempotency mechanism.

---

# 🛠️ Tech Stack

| Technology      | Purpose                            |
| --------------- | ---------------------------------- |
| Java            | Core application language          |
| Spring Boot     | Backend framework                  |
| Spring Data JPA | Database persistence               |
| H2 Database     | In-memory development database     |
| Maven           | Build and dependency management    |
| AES-256-GCM     | Authenticated symmetric encryption |
| RSA-OAEP        | Secure encryption of AES keys      |
| SHA-256         | Packet hashing / idempotency       |
| JUnit           | Automated testing                  |

---

# 📂 Project Structure

```text
UPI_Without_Internet-main/
│
├── pom.xml
├── mvnw
├── mvnw.cmd
├── README.md
│
└── src/
    │
    ├── main/
    │   │
    │   ├── java/
    │   │   └── com/demo/upimesh/
    │   │
    │   │       ├── UpiMeshApplication.java
    │   │
    │   │       ├── model/
    │   │       │   ├── Account.java
    │   │       │   ├── Transaction.java
    │   │       │   ├── MeshPacket.java
    │   │       │   └── PaymentInstruction.java
    │   │
    │   │       ├── crypto/
    │   │       │   ├── ServerKeyHolder.java
    │   │       │   └── HybridCryptoService.java
    │   │
    │   │       ├── service/
    │   │       │   ├── DemoService.java
    │   │       │   ├── MeshSimulatorService.java
    │   │       │   ├── VirtualDevice.java
    │   │       │   ├── IdempotencyService.java
    │   │       │   ├── SettlementService.java
    │   │       │   └── BridgeIngestionService.java
    │   │
    │   │       ├── controller/
    │   │       │   ├── ApiController.java
    │   │       │   └── DashboardController.java
    │   │
    │   │       └── config/
    │   │           └── AppConfig.java
    │   │
    │   └── resources/
    │       ├── application.properties
    │       └── templates/
    │           └── dashboard.html
    │
    └── test/
        └── java/
            └── IdempotencyConcurrencyTest.java
```

---

# 🚀 Getting Started

## Prerequisites

You need:

* Java 17 or higher
* Git

Check your Java installation:

```bash
java -version
```

---

# 📥 Clone the Repository

```bash
git clone https://github.com/aditya0za005-ux/UPI-Without-Internet.git
```

Navigate to the project:

```bash
cd UPI-Without-Internet/UPI_Without_Internet-main
```

---

# ▶️ Run the Application

## Windows

```bash
.\mvnw.cmd spring-boot:run
```

## macOS / Linux

```bash
./mvnw spring-boot:run
```

Once the application starts, open:

```text
http://localhost:8080
```

---

# 🖥️ Demo Flow

The dashboard allows you to demonstrate the complete lifecycle of an offline payment.

---

## Step 1 — Create a Payment

Choose:

* Sender
* Receiver
* Amount
* PIN

Click:

```text
📤 Inject into Mesh
```

The system:

```text
Create PaymentInstruction
        ↓
Generate Nonce
        ↓
Encrypt Payload
        ↓
Create MeshPacket
        ↓
Inject into Virtual Device
```

---

## Step 2 — Run Gossip

Click:

```text
🔄 Run Gossip Round
```

The packet propagates through the simulated network.

Multiple devices may now contain copies of the same packet.

---

## Step 3 — Bridge Upload

Click:

```text
📡 Bridges Upload to Backend
```

A device with internet connectivity uploads the encrypted packet.

The backend performs:

```text
Hash
 ↓
Idempotency Check
 ↓
Decrypt
 ↓
Freshness Check
 ↓
Settlement
 ↓
Ledger Update
```

---

## Step 4 — Observe Settlement

Check:

* 💰 Account balances
* 📜 Transaction ledger
* 📡 Mesh state

You can observe the payment lifecycle from creation to settlement.

---

# 🌐 API Reference

| Method | Endpoint             | Description                                |
| ------ | -------------------- | ------------------------------------------ |
| GET    | `/`                  | Interactive dashboard                      |
| GET    | `/api/server-key`    | Returns server public key                  |
| GET    | `/api/accounts`      | Returns account balances                   |
| GET    | `/api/transactions`  | Returns recent transactions                |
| GET    | `/api/mesh/state`    | Returns virtual device state               |
| POST   | `/api/demo/send`     | Creates and injects a payment              |
| POST   | `/api/mesh/gossip`   | Runs a gossip round                        |
| POST   | `/api/mesh/flush`    | Uploads bridge packets                     |
| POST   | `/api/mesh/reset`    | Resets the simulation                      |
| POST   | `/api/bridge/ingest` | Production-style bridge ingestion endpoint |
| GET    | `/h2-console`        | Opens H2 database console                  |

---

# 📦 Example Bridge Request

```http
POST /api/bridge/ingest
Content-Type: application/json

X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3
```

```json
{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-encrypted-payload"
}
```

Example response:

```json
{
  "outcome": "SETTLED",
  "packetHash": "a3f8c9...",
  "reason": null,
  "transactionId": 42
}
```

Possible outcomes:

```text
SETTLED
DUPLICATE_DROPPED
INVALID
```

---

# 🧪 Running Tests

Run all tests:

```bash
.\mvnw.cmd test
```

The project tests important security and concurrency behaviour.

---

## 🔐 Encryption Round Trip

Verifies:

```text
Encrypt → Decrypt → Original Data
```

---

## 🛡️ Tampered Packet Test

The test modifies encrypted data.

Expected behaviour:

```text
Tampered Ciphertext
        ↓
AES-GCM Authentication Failure
        ↓
INVALID
        ↓
No Settlement
```

---

## 🔁 Concurrent Idempotency Test

Multiple threads submit the same packet simultaneously.

Expected:

```text
1 × SETTLED
2 × DUPLICATE_DROPPED
```

The sender must only be charged once.

---

# 🏭 Production Architecture

The current implementation uses lightweight components to make the project easy to run locally.

A production architecture would look more like:

| Demo                     | Production                      |
| ------------------------ | ------------------------------- |
| H2                       | PostgreSQL                      |
| ConcurrentHashMap        | Redis                           |
| Generated RSA keys       | HSM / KMS                       |
| Virtual mesh             | BLE / Wi-Fi Direct              |
| Simulated accounts       | Banking infrastructure          |
| No bridge authentication | mTLS / signed certificates      |
| In-memory simulation     | Distributed infrastructure      |
| Local logs               | Structured logging + monitoring |

---

## Production-Style Architecture

```text
Mobile Devices
      │
      ▼
BLE / Wi-Fi Direct Mesh
      │
      ▼
Authenticated Bridge Nodes
      │
      ▼
API Gateway
      │
      ▼
Payment Ingestion Service
      │
      ├──────────────► Redis
      │                 Idempotency
      │
      ▼
Settlement Service
      │
      ├──────────────► PostgreSQL
      │
      ├──────────────► Fraud Detection
      │
      └──────────────► Banking / Payment Network
```

---

# ⚠️ Limitations of Offline Payments

This project intentionally demonstrates an important reality:

> **Transporting a payment offline is easier than guaranteeing that the payment can be settled.**

---

## ❌ Offline Balance Verification

The receiver cannot guarantee that the sender actually has sufficient funds.

The payment may travel successfully through the mesh but later fail during settlement.

---

## ❌ Offline Double Spending

A malicious sender could theoretically create multiple offline payment instructions before any of them reach the backend.

For example:

```text
Alice → Bob ₹500

Alice → Carol ₹500
```

If Alice only has ₹500 available, both payments cannot ultimately settle.

---

## ❌ Real Bluetooth Mesh Complexity

A production mobile implementation would need to deal with:

* Background execution restrictions
* Bluetooth permissions
* Battery consumption
* Device discovery
* Connection reliability
* Android/iOS platform limitations
* Intermittent connectivity

This project simulates these networking problems to focus on the backend and distributed-systems challenges.

---

# 🎯 What I Learned From Building This

This project explores concepts commonly found in fintech and distributed systems:

* Hybrid cryptography
* Authenticated encryption
* Idempotency
* Concurrent request handling
* Replay protection
* Database transactions
* Optimistic locking
* Distributed message delivery
* Eventually connected systems
* Fault-tolerant payment processing

---

# 🔮 Future Improvements

Potential next steps for the project:

* [ ] PostgreSQL instead of H2
* [ ] Redis-based distributed idempotency
* [ ] Dockerized deployment
* [ ] Real Android client
* [ ] Bluetooth Low Energy communication
* [ ] Device authentication
* [ ] Digital signatures
* [ ] Rate limiting
* [ ] Fraud detection rules
* [ ] Transaction status notifications
* [ ] Dead-letter handling for failed packets
* [ ] Observability and metrics
* [ ] Distributed tracing
* [ ] Deployment to cloud infrastructure

---

# 🧠 Interview Explanation

If someone asks:

> **"Explain your UPI Offline Mesh project."**

A strong answer would be:

> I built a Spring Boot-based simulation of an offline payment system where encrypted payment instructions can travel through a Bluetooth-style mesh network. Devices relay encrypted packets without being trusted to read or modify the transaction. Once any bridge device regains internet connectivity, it uploads the packet to the backend for settlement.
>
> The main engineering challenge was duplicate delivery because multiple devices can carry the same packet. I implemented idempotency using a SHA-256 hash of the ciphertext and an atomic claim operation, ensuring the payment is processed only once even when multiple bridge nodes submit it concurrently.
>
> The system also uses hybrid RSA-OAEP and AES-256-GCM encryption, replay protection using timestamps and nonces, transactional settlement, and optimistic locking for additional concurrency protection.

---

# 👨‍💻 Author

**Aditya Oza**

Java Backend Developer | Spring Boot | Distributed Systems | Fintech

---

## ⭐ If you found this project interesting

Consider giving the repository a star!

---

> **This project is not about making UPI magically work without the internet.**
>
> **It's about exploring what happens when payment instructions must survive an unreliable, disconnected world—and how cryptography, idempotency, and distributed-system design can make that possible.**
