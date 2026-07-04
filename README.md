
#  Three-Party Session Key Protocol (A–B–C)

A Python reference implementation of a key establishment protocol that derives a mutually agreed session key between three entities A, B, and C. The session key then secures a group chat with **Confidentiality, Integrity, Availability**, and **Non-Repudiation**.




---

## 🔄 Protocol Flow

The protocol establishes a mutually agreed session key between three participants (**A**, **B**, and **C**) through a trusted authentication server (**S**). Once the session key is established, all group communication is encrypted to ensure **Confidentiality**, **Integrity**, **Availability**, and **Non-Repudiation**.

---

## Phase 1: Authentication (Using RSA)

Each participant authenticates with the server before participating in the key establishment process.

### Entity A authenticates to Server S

```text
A → S : CA⟨⟨A⟩⟩, N_A
```

- A sends its certificate and a fresh nonce to the server.

```text
S → A : CA⟨⟨S⟩⟩, N_S, Sign_S(N_A)
```

- The server returns its certificate, a fresh nonce, and signs A's nonce using its private key.

```text
A → S : Sign_A(N_S)
```

- A signs the server's nonce to complete mutual authentication.

This process establishes secure temporary session keys (`K_AS`, `K_BS`, and `K_CS`) derived from the exchanged nonces.

The same authentication procedure is performed for **B ↔ S** and **C ↔ S**.

---

## Phase 2: Certificate Distribution

After successful authentication, the server securely distributes peer certificates.

### Server → A

```text
S → A : {CA⟨⟨B⟩⟩, CA⟨⟨C⟩⟩}AES-GCM(K_AS)
        + Sign_S(Hash(CA⟨⟨B⟩⟩, CA⟨⟨C⟩⟩))
```

- Sends B's and C's certificates encrypted using **AES-256-GCM**.
- Signs the hash of the certificates to ensure authenticity and integrity.

### Server → B

```text
S → B : {CA⟨⟨A⟩⟩, CA⟨⟨C⟩⟩}AES-GCM(K_BS)
        + Sign_S(Hash(CA⟨⟨A⟩⟩, CA⟨⟨C⟩⟩))
```

### Server → C

```text
S → C : {CA⟨⟨A⟩⟩, CA⟨⟨B⟩⟩}AES-GCM(K_CS)
        + Sign_S(Hash(CA⟨⟨A⟩⟩, CA⟨⟨B⟩⟩))
```

Each participant verifies the received certificates before proceeding.

---

## Phase 3: Key Contribution

Each participant generates a random key contribution that will be used to derive the shared group session key.

### Entity A

```text
A → S : {k_A}RSA(PK_B), {k_A}RSA(PK_C)
        + Sign_A(Hash(MSG))
```

- Generates a random key contribution `k_A`.
- Encrypts the contribution separately using **B's** and **C's** public RSA keys.
- Signs the hash of the message using A's private key.

The server forwards the encrypted contributions to the intended recipients.

```text
S → B : {k_A}RSA(PK_B) + Sign_S(Hash(MSG))

S → C : {k_A}RSA(PK_C) + Sign_S(Hash(MSG))
```

The same procedure is repeated for participants **B** and **C**.

---

## Phase 4: Session Key Derivation

After receiving all key contributions, every participant independently derives the same shared session key.

```text
K_ABC = HKDF(k_A || k_B || k_C)
```

Where:

- `HKDF` is the HMAC-based Key Derivation Function.
- `||` denotes concatenation of the three key contributions.

Since all participants possess the same inputs, each derives an identical session key without transmitting it directly.

---

## Phase 5: Secure Group Communication

Once the shared session key (`K_ABC`) has been established, all communication is protected using **AES-256-GCM**.

```text
A → S → B,C : {M1}AES-GCM(K_ABC)
              + Sign_A(Hash(M1))

B → S → A,C : {M2}AES-GCM(K_ABC)
              + Sign_B(Hash(M2))

C → S → A,B : {M3}AES-GCM(K_ABC)
              + Sign_C(Hash(M3))
```

Every transmitted message provides:

-  **Confidentiality** through AES-256-GCM encryption.
-  **Integrity** through authenticated encryption and digital signatures.
-  **Authentication** through certificate-based identity verification.
-  **Non-Repudiation** through RSA digital signatures.
-  **Availability** by maintaining authenticated communication between all participating entities.

---

### Security Objectives Achieved

| Security Property | Mechanism |
|-------------------|-----------|
| Confidentiality | AES-256-GCM Encryption |
| Integrity | SHA-256 Hashing + Digital Signatures |
| Authentication | CA-Issued RSA Certificates |
| Non-Repudiation | RSA Digital Signatures |
| Availability | Trusted Authentication Server & Secure Communication Protocol |

##  Project Structure

```
src/
  certificates.py   # Root CA & leaf cert generation/signing
  server.py         # Authenticates clients, relays messages, signals start_key
  client.py         # Auths to server, exchanges signed key shares, secures chat
tests/
  test_certificates.py
  test_protocol_sanity.py
```

---

##  Quickstart

### 1) Install
```bash
git clone https://github.com/SushankYerva/E2EKeyEstablishmentProtocol.git
cd three-party-session-key
python -m venv .venv
# Windows: .venv\Scriptsctivate
# macOS/Linux:
source .venv/bin/activate
pip install -r requirements.txt
```

### 2) Generate certificates
```bash
python -m src.certificates
```
This creates `certificates/` with `RootCA`, `ChatServer`, and `A/B/C` keypairs + JSON certs.

### 3) Run the server
```bash
python -m src.server
# [SERVER] Listening on 127.0.0.1:5000
```

### 4) Run three clients (in three terminals)
```bash
# Terminal 1
python -m src.client A
# Terminal 2
python -m src.client B
# Terminal 3
python -m src.client C
```

When all three authenticate, the server broadcasts `start_key`. Each client:
- fetches peer certificates,
- sends its **signed** key share (RSA),
- derives the common **group key**,
- and can send **AES-GCM** encrypted messages.

> You should see `Group key established.` on each client, then type messages to chat.

---



