# Auth & End-to-End Encryption

How a client (iOS app / CLI) authenticates with the server and how message content is
end-to-end encrypted. This is the consolidated, code-verified view; for adjacent detail see
[encryption.md](encryption.md) (encryption boundaries / on-wire encoding) and
[user-identity.md](user-identity.md) (identity model). File references point at the current
implementation in `packages/happy-app` and `packages/happy-server`.

## The core idea: two independent key universes

The design separates **identity** from **confidentiality**, each rooted in a secret the other
side never sees. The server proves who you are and routes your ciphertext, but never holds a key
that can read message content.

```
CLIENT (iOS app / CLI) -- holds the only copy of the root secret
+--------------------------------------------------------------------+
|  master secret (32 bytes)   generated on device, never sent out    |
|        |                                                           |
|        +-- crypto_sign_seed_keypair() ------> Ed25519 IDENTITY key |
|        |                                       (used for auth)      |
|        |                                                           |
|        +-- deriveKey('Happy EnCoder',['content'])                  |
|        |        \-- crypto_box_seed_keypair() --> X25519 CONTENT    |
|        |                                          key (wraps keys)  |
|        |                                                           |
|        +-- per-session random AES-256 data key                     |
|                (stored server-side, sealed-box-wrapped to CONTENT) |
+--------------------------------------------------------------------+
       |                      |                        |
       | sign challenge       | AES-256-GCM(message)   | wrapped data key
       v                      v                        v
=========================== TRUST WALL ==============================
  server proves who you are + routes your ciphertext, but holds NO
  key that can read message content
====================================================================
       |                      |                        |
       v                      v                        v
+------------------------+   +-------------------------------------+
| SERVER secret          |   | Postgres  (opaque ciphertext only)  |
| HANDY_MASTER_SECRET    |   |   Account.publicKey(hex) = identity  |
|  |- signs bearer tokens|   |   messages / metadata / agentState   |
|  \- KeyTree encrypts   |   |   + wrapped data keys = base64 CT    |
|     3rd-party svc tokens|  |   (server CANNOT decrypt content)    |
+------------------------+   +-------------------------------------+
```

Load-bearing property: **the bearer token authorizes *transport*, not *reading*.** A compromised
server or a stolen token grants routing access to ciphertext only — content keys live solely on
the user's devices, derived from a secret the server never receives.

## End-to-end flow: link -> authenticate -> encrypt -> sync

```
PHASE 1 -- Device linking (first launch only; QR moves the secret E2E)
  New iOS:        gen EPHEMERAL box keypair (epk/esk)            [authQRStart.ts]
  New -> Server:  POST /v1/auth/account/request { epk }
  Server -> New:  state: requested        (stores pending req keyed by epk)
  New -> Old:     show QR(epk)
  Old:            resp = sealedBox(masterSecret -> epk)          [authApprove.ts]
  Old -> Server:  POST /v1/auth/account/response { epk, resp } + Bearer(old)
  New -> Server:  poll /v1/auth/account/request { epk }
  Server -> New:  state: authorized { token, resp }
  New:            masterSecret = decryptBox(resp, esk)           [authQRWait.ts]
  => server only ever saw ciphertext `resp`; it cannot read the secret

PHASE 2 -- Direct auth (every cold start / token refresh)
  New:            idKey = sign_seed(masterSecret)
                  challenge = random 32 bytes ; sig = sign(challenge)  [authChallenge.ts]
  New -> Server:  POST /v1/auth { publicKey, challenge, signature }
  Server:         verify sig ; upsert Account by publicKey(hex)       [authRoutes.ts:22-33]
  Server -> New:  { token }   (privacy-kit token signed by HANDY_MASTER_SECRET, cached 24h)

PHASE 3 -- Send an encrypted message
  New:            dataKey = session AES-256 key (random; wrapped copy stored once)
                  c = base64( 0x00 || AES-256-GCM(JSON(msg), dataKey) )  [encryptor.ts]
  New -> Server:  message { content: { t:'encrypted', c } } + Bearer token  (HTTP / Socket.IO)
  Server:         store `c` as opaque blob ; allocate seq ; fan out `update`
  Server -> Peer: update { new-message, content:{ t:'encrypted', c } }
  Peer:           unwrap dataKey with CONTENT private key (same masterSecret)
                  -> AES-GCM open -> Zod validate                  [sessionEncryption.ts:160]
```

Two distinct key types are in play, and they must not be confused:
- The **identity** keypair (Ed25519) is derived *directly* from the master secret and is used only
  to prove account ownership at `/v1/auth`.
- The **ephemeral** box keypair in Phase 1 is a throwaway, generated fresh per linking attempt,
  used only as the recipient for the sealed-box transfer of the master secret. It is not the
  account identity.

## User-facing linking flow (who scans what)

Counter-intuitive but important: **the device being added shows the QR; your existing
logged-in phone scans it.** A laptop has no camera pointed at the phone, so the terminal is the
screen and the phone is the scanner + key-holder.

```
  [ Your computer ]                 [ Your phone ]
  run `happy`, pick "mobile"        (already logged in; holds the account secret)
       |                                   |
  prints a QR in the terminal  ---scan-->  "Connect terminal" -> camera scans QR
  (happy://terminal?<pubkey>)              |
       |                            wraps the account secret/content key to the
       |                            terminal's pubkey; POSTs ciphertext to server
       v                                   |
  polls server, decrypts <-----------------+
  -> linked; `happy` now works
```

Steps (verified: `happy-cli/src/ui/auth.ts`, `happy-app/.../useConnectTerminal.ts`):
1. On the computer, run `happy` and choose **"mobile"**. The CLI generates a throwaway keypair
   and prints a QR (a `happy://terminal?<publicKey>` deep link). ("web" instead opens a browser
   auth URL — same result, no camera.)
2. On your already-logged-in phone, open **Connect terminal** and scan the QR.
3. The phone wraps the secret to the terminal's public key and POSTs the ciphertext; the CLI
   polls, decrypts, and is linked.

Same rule for adding a second phone or the web app: the *new* device shows the QR, your
*existing* phone scans it (the `/v1/auth/account/*` endpoints). The very first device has nobody
to scan from, so it generates the account secret itself.

**Least-privilege link (V1 vs V2):** the CLI advertises `supportsV2`. V1 transfers the full
master secret; V2 transfers only the derived content data key
(`encryptBox(0x00 || contentDataKey, …)`), so a linked terminal can read/write encrypted content
but is never handed the identity-signing secret. Prefer V2.

## Key derivation (verified in code)

- **Identity** (auth): `crypto_sign_seed_keypair(masterSecret)` — Ed25519, used directly. The
  **public-key hex IS the account** (`authRoutes.ts:29`); there are no passwords.
- **Content keypair** (key-wrapping): `deriveKey(masterSecret, 'Happy EnCoder', ['content'])` ->
  `crypto_box_seed_keypair(...)` — X25519 (`sources/sync/encryption/encryption.ts:17-20`).
  `deriveKey` is a BIP32-style HMAC-SHA512 key tree (`sources/encryption/deriveKey.ts`): root =
  `HMAC(seed, usage + ' Master Seed')`, children chain on the chain-code. Deterministic, so every
  device holding the master secret derives identical keys.
- **Per-session data key**: a random AES-256 key, stored server-side wrapped as
  `0x00 || sealedBox(dataKey -> contentPubKey)` (`encryption.ts:208`, "encrypt to ourselves").
  Any of the user's devices can unwrap it; the server cannot.
- **Messages / metadata / agent state**: `AES-256-GCM(JSON(item), dataKey)` with a `0x00` version
  byte, base64, carried on the wire as `{ t: 'encrypted', c }` (the `@slopus/happy-wire`
  `SessionMessageContentSchema`). Decrypt is followed by Zod `safeParse`
  (`sources/sync/encryption/sessionEncryption.ts:160`).
- **Blobs (image attachments)**: a separate key `deriveKey(dataKey, 'Happy Blobs', ['session'])`
  + NaCl secretbox — cryptographically isolated from the message key.
- **Server side, fully separate**: `HANDY_MASTER_SECRET` -> a `privacy-kit` KeyTree, used only to
  sign bearer tokens (`sources/app/auth/auth.ts`) and to encrypt third-party service tokens
  (GitHub/OpenAI/Anthropic/Gemini) at rest (`sources/modules/encrypt.ts`). It never touches user
  content.

## Primitives reference

| Concern                 | Primitive                                   | Where                                   |
|-------------------------|---------------------------------------------|-----------------------------------------|
| Identity / auth signing | Ed25519 (`crypto_sign_detached`)            | `auth/authChallenge.ts`, `authRoutes.ts`|
| Server token verify     | `tweetnacl.sign.detached.verify`            | `authRoutes.ts:22`                       |
| Bearer tokens           | privacy-kit persistent token (server secret)| `app/auth/auth.ts`                       |
| Secret handoff / wrap   | X25519 box (`crypto_box_easy`, eph+nonce)   | `encryption/libsodium.ts:8-34`           |
| Message content         | AES-256-GCM over JSON, `0x00` version byte  | `sync/encryption/encryptor.ts:81-125`    |
| Legacy content / blobs  | NaCl secretbox (`crypto_secretbox_easy`)    | `encryption/libsodium.ts:36-58`          |
| Service tokens at rest  | privacy-kit KeyTree (server secret)         | `modules/encrypt.ts`                     |

## The pattern (if adopting this)

1. One root secret per user, born on-device, never transmitted — identity and content keys both
   derive from it (identity directly, content via a labeled KDF path).
2. Auth = prove possession of the identity key -> server issues a bearer token bound to
   `account = identity public key`. The token authorizes transport/routing only.
3. Confidentiality = per-record AES-256-GCM key, sealed-box-wrapped to the user's content public
   key, so any of their devices unwrap it and the server cannot.
4. Multi-device = ephemeral sealed-box handoff of the root secret; the server brokers ciphertext
   only.
5. Keep the server's key universe separate (its own secret for token signing + encrypting any
   third-party credentials at rest).

## Caveats to weigh before reusing

- **The challenge is client-generated random, not a server nonce.** `authChallenge.ts` makes up 32
  random bytes and signs them; `/v1/auth` only verifies the signature. This proves key possession
  but is **not replay-resistant on its own** — anti-replay rests on TLS + bearer-token secrecy.
  Issue a server-side nonce if you need freshness guarantees. (Upside of this model: a
  replayed/stolen token still cannot read content.)
- **AES-GCM nonce handling lives in `sources/encryption/aes.ts` (`encryptAESGCMString`)** and is
  not covered here — GCM is catastrophic on nonce/IV reuse, so confirm a fresh random 96-bit IV
  per encryption before relying on that path. There is also a flagged AES/UTF-8 edge-case comment
  in `encryptor.ts:8`.
- **The legacy `SecretBoxEncryption` path** (NaCl secretbox with the master secret directly, no
  per-record key) exists only for pre-migration sessions. For a greenfield build, start at
  per-record AES keys wrapped to a content keypair — cleaner isolation and rotation.
