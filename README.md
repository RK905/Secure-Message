# Secure-Message
No HandShake , No Chat

Build an MVP secure messaging app called **Handshake**.

The app is a local-first, serverless, text-only secure messenger. Users can only message people they have verified in person by scanning QR codes. The server must not store permanent accounts, contacts, group rosters, plaintext messages, or permanent message history.

The core product idea:

**“No handshake, no chat.”**

Messages are encrypted on the sender’s device, temporarily relayed through AWS serverless infrastructure, fetched by the recipient, decrypted locally, and deleted from the server after delivery or expiration.

## Core architecture

Use this architecture:

* Mobile app: React Native with Expo
* Backend: AWS Lambda using the Serverless Framework
* WebSocket layer: AWS API Gateway WebSocket API
* HTTP API layer: AWS API Gateway HTTP API
* Temporary message storage: DynamoDB with TTL
* Local storage: encrypted device storage
* No PostgreSQL
* No permanent message database
* No permanent account database for MVP
* No cloud message history
* No media, no audio, no video, no files

## Important privacy model

The server is not a chat backend.

The server is only an **ephemeral encrypted mailbox relay**.

The server may temporarily store ciphertext in DynamoDB so offline users can fetch pending messages, but:

1. Messages must be encrypted before leaving the sender’s device.
2. The server must never receive plaintext.
3. DynamoDB must store ciphertext only.
4. Messages must have a short expiration time.
5. The API must never return expired messages, even if DynamoDB TTL has not physically deleted them yet.
6. Messages must be explicitly deleted after recipient acknowledgment.
7. DynamoDB TTL is only backup cleanup, not the primary deletion mechanism.
8. No message content should appear in logs.
9. No contact lists should be stored on the server.
10. No group rosters should be stored on the server.

## Product scope

Build only:

* Text messaging
* In-person QR verification
* 1-to-1 chats
* Small group chats
* Local encrypted message history
* Ephemeral server-side delivery queue
* WebSocket live delivery
* HTTP fallback fetch
* Short-TTL offline delivery
* Delete-after-ack
* Key-change warnings
* Local-only contacts
* Local-only groups

Do not build:

* Images
* Video
* Audio
* Files
* Voice notes
* Stickers
* Stories
* Channels
* Bots
* Public search
* Username directory
* Cloud backups
* Web app
* Multi-device sync
* Server-side contacts
* Server-side message history

## Human verification model

Every device generates its own identity key pair.

The private identity key must never leave the device.

The user’s QR code should contain only safe public/routing information:

```json
{
  "protocolVersion": 1,
  "displayName": "Rayen",
  "publicIdentityKey": "...",
  "identityFingerprint": "...",
  "mailboxId": "...",
  "writeToken": "...",
  "createdAt": "..."
}
```

The QR code must not contain:

* private keys
* message keys
* local database keys
* plaintext chat history

When two users meet in person:

1. Alice scans Bob’s QR code.
2. Bob scans Alice’s QR code.
3. Each app stores the other person locally as a verified contact.
4. The chat unlocks only after both users have verified each other.
5. If a contact’s identity key changes, the chat locks again until re-verified.

User-facing language:

* “Chat locked.”
* “Verify in person to unlock.”
* “Handshake complete.”
* “Identity key changed. Verify again before chatting.”
* “Only scan this code if you are physically with this person.”

## Local app data

Store this only on the device:

* private identity key
* user profile
* verified contacts
* contact public keys
* contact fingerprints
* contact mailbox IDs
* contact write tokens
* group rosters
* group verification status
* message history
* outbound retry queue
* blocked contacts
* local app settings

Use encrypted local storage.

For React Native / Expo, create an abstraction like:

```ts
SecureLocalStore.set(key, value)
SecureLocalStore.get(key)
SecureLocalStore.delete(key)
```

Private keys should use the strongest available platform storage, such as iOS Keychain and Android Keystore-compatible secure storage.

## Backend goal

Build a serverless AWS backend using the Serverless Framework.

The backend should provide:

1. HTTP endpoint to send encrypted messages
2. HTTP endpoint to fetch pending messages
3. HTTP endpoint to acknowledge and delete delivered messages
4. WebSocket connect route
5. WebSocket disconnect route
6. WebSocket subscribe route
7. WebSocket live notification route
8. DynamoDB TTL-based temporary storage
9. DynamoDB table for active WebSocket connections
10. No permanent user database

## Serverless Framework structure

Create a backend folder like:

```txt
backend/
  serverless.yml
  package.json
  src/
    handlers/
      sendMessage.ts
      fetchMessages.ts
      ackMessage.ts
      websocketConnect.ts
      websocketDisconnect.ts
      websocketSubscribe.ts
      websocketSend.ts
    lib/
      dynamo.ts
      auth.ts
      validation.ts
      expiry.ts
      logging.ts
      websocket.ts
```

Use TypeScript.

## DynamoDB tables

Create two DynamoDB tables.

### 1. EphemeralMessages

This table stores temporary encrypted message packets only.

Schema:

```txt
Table name: HandshakeEphemeralMessages

Partition key:
- mailboxId

Sort key:
- messageId

Attributes:
- mailboxId
- messageId
- ciphertext
- encryptionHeader
- createdAt
- expiresAt
- ttl
- leaseUntil
- deliveryAttemptCount
- maxDeliveryAttempts
- messageSize
```

Rules:

* `ttl` must be a Unix epoch timestamp in seconds.
* `expiresAt` must also be stored and checked by application logic.
* Never return items where `expiresAt <= now`.
* Delete messages after successful acknowledgment.
* Delete messages after expiration.
* Reject messages with TTL longer than the configured max.
* Do not store sender name.
* Do not store recipient name.
* Do not store group ID.
* Do not store plaintext message body.
* Do not log ciphertext.

Recommended default TTLs:

```txt
Strict mode: 15 minutes
Balanced mode: 1 hour
Maximum allowed: 24 hours
```

### 2. WebSocketConnections

This table stores temporary live connection state.

Schema:

```txt
Table name: HandshakeWebSocketConnections

Partition key:
- mailboxId

Sort key:
- connectionId

Attributes:
- mailboxId
- connectionId
- connectedAt
- ttl
```

Rules:

* Connection records should expire automatically.
* Delete connection records on disconnect.
* Do not store user names.
* Do not store contacts.
* Do not store group information.

## HTTP API endpoints

Create these endpoints.

### Send encrypted message

```txt
POST /v1/mailboxes/{mailboxId}/messages
```

Headers:

```txt
Authorization: Bearer {writeToken}
```

Body:

```json
{
  "messageId": "random-256-bit-id",
  "ciphertext": "...",
  "encryptionHeader": {
    "version": 1,
    "algorithm": "placeholder-e2ee-v1",
    "nonce": "...",
    "senderEphemeralPublicKey": "..."
  },
  "expiresInSeconds": 3600
}
```

Behavior:

1. Validate mailbox ID.
2. Validate write token.
3. Validate message size.
4. Validate `expiresInSeconds`.
5. Compute `expiresAt`.
6. Store ciphertext in DynamoDB.
7. Try to notify active WebSocket connections for that mailbox.
8. Return success without exposing sensitive details.

Response:

```json
{
  "ok": true,
  "messageId": "...",
  "expiresAt": 1234567890
}
```

### Fetch pending messages

```txt
GET /v1/mailboxes/{mailboxId}/messages
```

Headers:

```txt
Authorization: Bearer {readToken}
```

Behavior:

1. Validate read token.
2. Query DynamoDB by mailbox ID.
3. Filter out expired messages in application code.
4. Lease returned messages briefly.
5. Return ciphertext packets only.
6. Do not delete yet.

Response:

```json
{
  "messages": [
    {
      "messageId": "...",
      "ciphertext": "...",
      "encryptionHeader": {},
      "createdAt": 1234567000,
      "expiresAt": 1234567890
    }
  ]
}
```

### Acknowledge message

```txt
POST /v1/mailboxes/{mailboxId}/messages/{messageId}/ack
```

Headers:

```txt
Authorization: Bearer {readToken}
```

Behavior:

1. Validate read token.
2. Delete message from DynamoDB immediately.
3. Return success.

Response:

```json
{
  "ok": true,
  "deleted": true
}
```

## WebSocket routes

Create these WebSocket routes with API Gateway and Lambda.

### $connect

On connect:

1. Validate the client.
2. Accept connection.
3. Do not yet subscribe to mailbox unless mailbox info is provided securely.

### $disconnect

On disconnect:

1. Remove connection ID from WebSocketConnections table.

### subscribe

Client sends:

```json
{
  "type": "subscribe",
  "mailboxId": "...",
  "readToken": "..."
}
```

Behavior:

1. Validate read token.
2. Store `mailboxId -> connectionId` in WebSocketConnections.
3. Return subscription success.

### new message notification

When a message arrives and recipient has active WebSocket connection, send:

```json
{
  "type": "new_message_available",
  "mailboxId": "...",
  "count": 1
}
```

Do not send plaintext over WebSocket.

For MVP, the server may either:

1. send only the notification and let the app call fetch, or
2. send the ciphertext packet directly over WebSocket.

Prefer option 1 first for simpler reliability.

## Message delivery behavior

Support three recipient states.

### Recipient online

1. Sender posts encrypted message.
2. Lambda stores ciphertext in DynamoDB.
3. Lambda sends WebSocket event: `new_message_available`.
4. Recipient fetches message.
5. Recipient decrypts locally.
6. Recipient sends ack.
7. Lambda deletes message.

### Recipient offline

1. Sender posts encrypted message.
2. Lambda stores ciphertext with TTL.
3. Recipient later opens app.
4. App fetches pending messages.
5. App decrypts locally.
6. App sends ack.
7. Lambda deletes message.

### Message expires

1. Message reaches `expiresAt`.
2. API must stop returning the message.
3. Background cleanup may delete it.
4. DynamoDB TTL eventually removes it if not already deleted.

## Lease and acknowledgment

Implement a lease system to avoid message loss.

When messages are fetched:

* Set `leaseUntil = now + 30 seconds`.
* Do not return leased messages to another request unless lease has expired.
* Do not delete on fetch.
* Delete only after ack.
* If app crashes before ack, the message can be fetched again until expiration.

## Encryption service in the mobile app

Create an encryption abstraction.

Files:

```txt
app/src/crypto/
  identity.ts
  fingerprint.ts
  session.ts
  messageCrypto.ts
  groupCrypto.ts
```

Functions:

```ts
generateIdentityKeyPair()
getPublicIdentityKey()
getIdentityFingerprint(publicKey)
encryptMessage(plaintext, recipientPublicKey)
decryptMessage(ciphertext, privateKey)
createSessionWithVerifiedContact(contactId)
verifyIdentityFingerprint(contactId, fingerprint)
detectIdentityKeyChange(contactId, newPublicKey)
```

For the MVP, placeholder encryption may be used only if clearly marked as unsafe for production.

Add comments saying:

```txt
WARNING: This placeholder encryption is not production-grade.
Replace with an audited E2EE protocol before real users.
```

But still structure the app so production cryptography can be swapped in later.

Do not invent custom cryptography for production.

## 1-to-1 chat flow

Screens:

1. Welcome
2. Create local profile
3. My QR code
4. QR scanner
5. Verification confirmation
6. Local contacts
7. Locked chat
8. Unlocked chat
9. Key changed warning
10. Settings

Flow:

1. User creates local profile.
2. App generates local identity key pair.
3. App generates mailbox ID, read token, and write token.
4. App displays QR code with public identity and write routing info.
5. Another user scans QR code in person.
6. Contact is saved locally.
7. If both users scan each other, chat unlocks.
8. Messages are encrypted locally and sent to the contact’s mailbox.
9. Received messages are decrypted locally and stored locally.

## Group chat flow

Groups must be local-only.

The server should not know groups exist.

Group rules:

1. Max 10 members for MVP.
2. Every member must verify every other member in person.
3. Group roster is stored locally on each device.
4. Group messages are sent as separate encrypted packets to each member’s mailbox.
5. New members cannot read old messages.
6. Removed members cannot read future messages.
7. When membership changes, future messages use a new local group epoch.

Group message sending:

If Alice sends to a group with Bob, Chris, and Dana:

```txt
Alice encrypts one packet for Bob.
Alice encrypts one packet for Chris.
Alice encrypts one packet for Dana.
Alice sends each packet to each person’s mailbox.
```

The server should only see independent encrypted mailbox deliveries.

Do not store:

* group name on server
* group ID on server
* group roster on server
* group message history on server

## Local group screens

Build:

1. Create group
2. Add verified contacts
3. Group verification checklist
4. Locked group screen
5. Unlocked group chat
6. Re-verification required screen

User-facing copy:

* “Group locked until everyone verifies each other.”
* “Everyone has completed the handshake.”
* “New member added. Re-verification required.”
* “This member can only read future messages.”

## Security requirements

Implement these requirements:

1. No plaintext messages leave the device.
2. No private keys leave the device.
3. No permanent message storage on server.
4. No permanent account database.
5. No contact graph on server.
6. No group database on server.
7. DynamoDB stores ciphertext only.
8. Messages expire quickly.
9. API must enforce expiry before returning messages.
10. Messages delete immediately after acknowledgment.
11. DynamoDB TTL is only backup cleanup.
12. Logs must never contain plaintext or ciphertext.
13. Push notifications, if added later, must never include message content.
14. Key changes must lock chats.
15. Users must re-verify in person after key changes.
16. Rate-limit sends, fetches, and subscriptions.
17. Limit message size.
18. Limit mailbox abuse with write tokens.
19. Rotate mailbox IDs and tokens if compromised.
20. Use least-privilege IAM permissions.

## Abuse and spam prevention

Because there are no accounts, use these protections:

* mailbox write tokens
* per-mailbox rate limits
* per-IP rate limits
* max message size
* max TTL
* max pending messages per mailbox
* reject duplicate message IDs
* reject malformed packets
* allow local blocking
* allow mailbox rotation

If a user blocks someone locally:

1. Delete their contact locally.
2. Stop accepting/decrypting their messages.
3. Optionally rotate mailbox ID and write token.
4. Show the user a new QR code.

## Serverless Framework requirements

Create a `serverless.yml` that defines:

* AWS provider
* Node.js runtime
* HTTP API routes
* WebSocket API routes
* Lambda functions
* DynamoDB tables
* TTL configuration
* IAM permissions
* environment variables

Environment variables:

```txt
EPHEMERAL_MESSAGES_TABLE
WEBSOCKET_CONNECTIONS_TABLE
MAX_MESSAGE_TTL_SECONDS
DEFAULT_MESSAGE_TTL_SECONDS
MAX_MESSAGE_SIZE_BYTES
MESSAGE_LEASE_SECONDS
```

Use least-privilege IAM.

Each Lambda should only have the DynamoDB permissions it needs.

## Logging rules

Create a safe logger.

The logger must never log:

* plaintext
* ciphertext
* private keys
* public keys unless explicitly in debug mode
* tokens
* full mailbox IDs

Logs may include:

* request ID
* operation type
* success/failure
* truncated mailbox hash
* error category
* timestamp

## README requirements

Create a README with:

1. Product overview
2. Architecture diagram in text
3. Local setup instructions
4. AWS deployment instructions using Serverless Framework
5. Environment variables
6. Security model
7. What the server stores
8. What the server never stores
9. DynamoDB TTL caveat
10. Known MVP limitations
11. What must be replaced before production
12. Security audit checklist

## MVP limitations to document

Document these clearly:

* Placeholder encryption is not production safe.
* DynamoDB TTL is not instant deletion.
* Server can still see metadata such as IP, timing, mailbox ID, and packet size.
* No multi-device support.
* No account recovery.
* Lost device means lost keys and local message history.
* If a user screenshots or copies a message, the app cannot prevent that fully.
* If a recipient receives a message, the sender cannot truly make them unread it.
* Push notifications are not implemented in v1.
* Offline delivery only works until TTL expires.

## Final deliverable

Build a working MVP with:

1. React Native Expo mobile app
2. Serverless Framework backend
3. API Gateway HTTP endpoints
4. API Gateway WebSocket endpoints
5. Lambda handlers
6. DynamoDB ephemeral message table with TTL
7. DynamoDB WebSocket connection table with TTL
8. QR-based in-person verification
9. Local-only contacts
10. Local-only groups
11. 1-to-1 encrypted text message flow
12. Small group encrypted text message flow
13. Fetch, decrypt, and ack behavior
14. Delete-after-ack behavior
15. Expiry enforcement in application code
16. Clear security warnings
17. README and deployment instructions

Prioritize privacy architecture, clean code, and working message delivery over fancy UI.

The MVP should prove this concept:

**A secure messenger where trust is created in person, messages live locally, and the server only acts as a short-lived encrypted delivery relay.**
