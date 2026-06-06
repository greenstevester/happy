# Implementation Guide: E2E Auth & Encryption (portable)

A stack-agnostic spec for building the same authentication + end-to-end encryption mechanism in
another project. It restates the design from [auth-and-encryption.md](auth-and-encryption.md) as
build instructions, with concrete primitives, endpoint contracts, milestones, and acceptance
tests. Copy this file into the target project and adapt the primitives to its language — every
primitive below has a binding in every mainstream stack (libsodium, WebCrypto, Go x/crypto, etc.).

Diagrams are ASCII so they render anywhere.

---

## 0. What you are building

A system where:
- A **single 32-byte root secret per user** is generated on-device and **never sent to the
  server**.
- The server **authenticates** users and **routes/stores** their data, but **cannot read message
  content** — it only ever holds ciphertext.
- Any number of the user's devices can be **linked** (each gets read/write access to content)
  by transferring the secret device-to-device, brokered but never readable by the server.

Security properties to preserve (these are the point — don't break them):

| Property                  | How it's achieved                                                        |
|---------------------------|--------------------------------------------------------------------------|
| Server can't read content | Content keys derive from the root secret, which the server never receives |
| Auth ≠ read access        | The bearer token authorizes transport only; it is not a content key       |
| Multi-device              | Root secret transferred via sealed box between devices                    |
| Tamper-evident            | AEAD (AES-256-GCM) auth tag on every record                               |
| Key separation            | Distinct derived keys for identity, content, and blobs                    |

---

## 1. Cryptographic primitives

Pick the equivalent in your stack. The reference uses libsodium (clients) + tweetnacl (server
verify).

| Purpose                  | Primitive                                  | Notes                                  |
|--------------------------|--------------------------------------------|----------------------------------------|
| Identity / signing       | Ed25519                                    | seed = the 32-byte root secret         |
| Key derivation (KDF)     | HKDF-SHA-256 **or** HMAC-SHA-512 key tree  | distinct `info`/label per derived key  |
| Key wrapping / transfer  | X25519 sealed box (ephemeral sender)       | `crypto_box_seal` or manual eph+box     |
| Content encryption       | AES-256-GCM                                 | random 96-bit nonce **per message**     |
| Bearer token             | server-signed token (Ed25519/HMAC/JWT)     | signed with a server-only secret        |

Rules:
- **Never reuse an AES-GCM nonce under the same key.** Generate 12 fresh random bytes each time.
- The **root secret is a CSPRNG 32 bytes**, generated on the first device. Treat it like a seed
  phrase: back it up, never transmit it in cleartext, never log it server-side.

---

## 2. The root secret and key hierarchy

Everything derives from one secret. Derive with labeled paths so keys are domain-separated.

```
root secret (32 bytes, on-device only)
  |
  |-- (used directly as Ed25519 seed) ----------> IDENTITY keypair
  |        account id = hex(identity public key)        (auth: sign challenges)
  |
  |-- KDF(root, "content")  --> X25519 seed -----> CONTENT keypair
  |        (wraps/unwraps per-record data keys; "encrypt to yourself")
  |
  |-- KDF(root, "blobs")    ----------------------> blob master key (binary attachments)
  |
  '-- per record: random 32-byte DATA KEY (AES-256)
           stored on server as: 0x00 || sealedBox(DATA KEY -> CONTENT public key)
```

- **Identity keypair**: Ed25519 from the root secret *directly*. Its public key (hex) is the
  account primary key. No passwords anywhere.
- **Content keypair**: X25519 from a *derived* subkey (`KDF(root, "content")`). Used only as the
  recipient when wrapping per-record data keys. Any device with the root secret can unwrap; the
  server cannot.
- **Per-record data key**: a fresh random AES-256 key per session/conversation/object. The actual
  message bytes are encrypted with this; the data key itself is wrapped to the content public key
  and stored alongside the record.
- Derive blob/attachment keys from the record's data key (`KDF(dataKey, "blob")`) so binary
  attachments are cryptographically isolated from message text.

> Greenfield recommendation: skip any "legacy" mode that encrypts directly with the root secret.
> Always go through per-record data keys — it gives you per-object isolation and rotation for free.

---

## 3. Server data model (minimum)

```
account            ( id, public_key_hex UNIQUE, created_at, updated_at )
auth_link_request  ( id, request_public_key UNIQUE, response_ciphertext NULL,
                     responder_account_id NULL, supports_v2 BOOL, created_at )
record             ( id, account_id, seq, wrapped_data_key, created_at )   # session/conversation
message            ( id, record_id, seq, content_ciphertext, created_at )  # { "t":"encrypted","c": "<b64>" }
```

- All `*_ciphertext` / `content_ciphertext` columns are **opaque base64 strings**. The server
  reads/writes them but never decrypts.
- `seq` is a monotonic counter (per-account or per-record) used for ordering and delta sync.
- Store `wrapped_data_key` once per record; reuse it for every message in that record.

---

## 4. HTTP API contract

All bodies are JSON; binary fields are base64. Two groups: **direct auth** (a device that already
holds the root secret) and **device linking** (move the secret to a new device).

### 4.1 Direct auth — prove identity, get a token

```
POST /v1/auth
  body:   { publicKey, challenge, signature }   # all base64; Ed25519
  server: verify Ed25519 signature of `challenge` by `publicKey`
          upsert account WHERE public_key_hex = hex(publicKey)
          issue bearer token bound to account.id
  200:    { token }
  401:    { error: "Invalid signature" }
```

### 4.2 Device linking — requester side (the new device)

```
POST /v1/auth/request
  body:   { publicKey, supportsV2 }     # requester's EPHEMERAL X25519 public key
  server: upsert auth_link_request keyed by request_public_key
          if already answered -> return token + response, else "requested"
  200:    { state: "requested" }
        | { state: "authorized", token, response }   # response = ciphertext to decrypt

GET /v1/auth/request/status?publicKey=...
  200:    { status: "not_found" | "pending" | "authorized", supportsV2 }
```

The requester **polls** `POST /v1/auth/request` (or the status endpoint) until `authorized`.

### 4.3 Device linking — approver side (existing logged-in device)

```
POST /v1/auth/response          # requires Authorization: Bearer <approver token>
  body:   { publicKey, response }       # publicKey = the requester's ephemeral pubkey
                                        # response  = sealedBox(secret -> requester pubkey), base64
  server: find auth_link_request by publicKey; if unanswered,
          set response_ciphertext = response, responder_account_id = caller.account
  200:    { success: true }
```

> Provide a parallel `/v1/auth/account/*` trio if you want to distinguish "link a CLI/terminal"
> from "link another app instance." The mechanics are identical; only the table/label differ.

---

## 5. Encrypted content wire format

Every encrypted value on the wire is the same envelope:

```
{ "t": "encrypted", "c": "<base64>" }
```

where the bytes inside `c` are:

```
c = base64( version_byte(0x00) || AES-256-GCM( key = data_key,
                                               nonce = random 12 bytes,
                                               plaintext = utf8(JSON.stringify(value)) ) )
```

- The leading `0x00` is a version byte — lets you evolve the scheme later. Reject anything whose
  first byte you don't recognize.
- Your AEAD output must include the nonce and auth tag (store `nonce || ciphertext || tag`, or
  whatever your library's sealed format is — just be consistent on both ends).
- **After decrypting, validate the JSON against a schema** (Zod/JSON-Schema/your equivalent)
  before trusting it. Decryption proves integrity; schema validation proves shape.

---

## 6. Client responsibilities

### 6.1 First device — create an account
1. Generate a random 32-byte **root secret** (CSPRNG). Persist it in the OS keystore/secure
   storage. Offer the user a backup (it is the only way to recover the account).
2. Authenticate (6.2) — the `upsert` creates the account on first contact.

### 6.2 Authenticate — get a token (every cold start / token expiry)
1. `identityKey = Ed25519_from_seed(rootSecret)`.
2. `challenge = random 32 bytes`; `signature = Ed25519_sign(challenge, identityKey.private)`.
3. `POST /v1/auth { publicKey, challenge, signature }` -> store the returned `token`.
4. Send the token as `Authorization: Bearer <token>` on HTTP and on your socket handshake.

> See pitfall (10.1): this challenge is client-generated. If you need replay resistance at the
> auth endpoint, add a server-issued nonce.

### 6.3 Link a new device

```
NEW DEVICE (requester)                     EXISTING DEVICE (approver, holds rootSecret + token)
1. eph = X25519 keypair (throwaway)
2. POST /v1/auth/request { eph.pub }
3. render eph.pub as a QR /
   deep link the OTHER device can read --> 4. scan/parse eph.pub from the QR
                                           5. payload = rootSecret   (V1)
                                                      | KDF(root,"content") only  (V2, preferred)
                                           6. response = sealedBox(payload -> eph.pub)
                                           7. POST /v1/auth/response { eph.pub, response } + Bearer
8. poll POST /v1/auth/request { eph.pub }
9. on "authorized": payload = sealedBoxOpen(response, eph.priv); store token + payload
```

Direction rule: **the device being added displays the QR; the already-authenticated device scans
it** (the scanner is the one with a camera and the secret).

V2 least-privilege: transfer only the derived content key, not the root secret, so a linked
device can read/write content but can't mint new identity signatures. Recommended default.

### 6.4 Encrypt / decrypt content
- **Send**: ensure the record has a `data_key` (generate + wrap + upload once). For each message:
  `c = encrypt(value, data_key)`; POST `{ content: { t:"encrypted", c } }` with the bearer token.
- **Receive**: get `wrapped_data_key` for the record, unwrap it with your content private key,
  then `value = decrypt(c, data_key)`; schema-validate.
- Cache unwrapped data keys in memory per record; never persist them in plaintext.

---

## 7. Server responsibilities — and what it must NEVER do

Do:
- Verify Ed25519 signatures; issue/verify bearer tokens (sign with a **server-only** secret, e.g.
  `SERVER_MASTER_SECRET`); cache verified tokens with a TTL.
- Store ciphertext blobs, allocate `seq`, fan out updates to the right subscribers.
- Keep a **separate** key hierarchy (derived from the server secret) **only** for encrypting any
  third-party credentials *you* hold at rest (OAuth tokens, vendor API keys). This never touches
  user content.
- Make writes **idempotent** (clients retry). Use a transaction wrapper with retry on
  serialization failure for multi-row writes; emit socket events only after commit.

Never:
- Never receive, store, or log the user's root secret or any content data key in plaintext.
- Never try to decrypt `content_ciphertext` / `wrapped_data_key` — you don't have the keys, by
  design.
- Never mix the two key universes (server secret vs user root secret).

---

## 8. Build order (milestones)

```
M1  Crypto lib chosen; round-trip unit tests: sign/verify, sealedBox open, AES-GCM encrypt/decrypt
M2  POST /v1/auth + account upsert + bearer tokens; client cold-start auth works
M3  Content path: per-record data key -> wrap -> store; encrypt/decrypt a message end-to-end
M4  Device linking: request/approve/poll endpoints + QR display/scan; second device decrypts M3's data
M5  Realtime fan-out (socket) with bearer auth; seq-ordered delta sync
M6  Hardening: V2 least-privilege link, token TTL/revocation, schema validation, nonce audit
```

Each milestone is independently testable; don't start M3 before M1's crypto round-trips pass.

---

## 9. Acceptance tests (must pass)

1. **Cross-device read**: two clients seeded with the *same* root secret — client A encrypts a
   message, client B decrypts it to the identical plaintext.
2. **Server blindness**: dump the DB; confirm no endpoint and no stored column yields plaintext
   without a client key. (Try to decrypt server-side and assert you can't.)
3. **Auth**: a valid signed challenge returns a token; a tampered signature returns 401; the token
   authorizes a content write.
4. **Linking**: a fresh device with no secret completes the request→approve→poll dance and ends up
   able to decrypt pre-existing messages. The server only ever saw ciphertext `response`.
5. **Tamper detection**: flip one byte of a stored `c`; decryption fails (GCM tag) rather than
   returning garbage.
6. **Nonce uniqueness**: encrypt the same plaintext twice; assert different ciphertext (proves a
   fresh nonce per call).

---

## 10. Pitfalls & hardening

1. **Client-generated challenge ≠ replay-resistant.** The reference signs random bytes the client
   made up, so `/v1/auth` proves key possession but not freshness; anti-replay rests on TLS +
   token secrecy. If that's not enough for your threat model, issue a server nonce
   (`GET /v1/auth/nonce` → sign *that*).
2. **AES-GCM nonce reuse is catastrophic.** Audit that every encryption draws a fresh 96-bit
   random nonce. Consider a key-commitment scheme if you encrypt very high volumes per key.
3. **Token lifetime & revocation.** Cache verified tokens with a TTL; provide a way to invalidate
   all tokens for an account (e.g., on device unlink). Token theft grants transport access, not
   content — keep it that way.
4. **Key separation.** Identity seed, content seed, and blob keys must be distinct derivations.
   Don't reuse one key across roles.
5. **Prefer V2 linking** (transfer the content key, not the root secret) so linked devices are
   least-privilege.
6. **Backup/recovery UX.** The root secret is the account. Make backup deliberate and make "I lost
   all devices" a conscious, documented dead-end (or build a separate recovery escrow — out of
   scope here).
7. **TLS still required.** E2E encryption protects content confidentiality; you still need TLS for
   metadata, tokens, and the linking handshake.

---

## Reference implementation (this repo)

- Identity / challenge: `packages/happy-app/sources/auth/authChallenge.ts`, server
  `packages/happy-server/sources/app/api/routes/authRoutes.ts`.
- Linking (requester / approver): `authQRStart.ts`, `authQRWait.ts`, `authApprove.ts`,
  `packages/happy-app/sources/hooks/useConnectTerminal.ts`; CLI side `happy-cli/src/ui/auth.ts`.
- Key hierarchy + wrapping: `packages/happy-app/sources/sync/encryption/encryption.ts`,
  `sources/encryption/deriveKey.ts`, `sources/encryption/libsodium.ts`.
- Content AEAD: `sources/sync/encryption/encryptor.ts` (`AES256Encryption`).
- Server token + service-token KeyTree: `happy-server/sources/app/auth/auth.ts`,
  `sources/modules/encrypt.ts`.
- Wire envelope: `@slopus/happy-wire` (`SessionMessageContentSchema`).
