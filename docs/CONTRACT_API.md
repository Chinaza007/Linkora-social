# Linkora API Reference

> **Scope:** HTTP request authentication for the indexer REST API and a source-based
> [contract event schema reference](#10-contract-event-schema-reference). For contract
> functions and storage, see `packages/contracts` and the README API table.

---

## Table of Contents

1. [Stellar HTTP Authentication (v1)](#1-stellar-http-authentication-v1)
2. [The Signed Message](#2-the-signed-message)
3. [Canonical Path](#3-canonical-path)
4. [Body Hash](#4-body-hash)
5. [The Authorization Header](#5-the-authorization-header)
6. [Verification Order and Error Codes](#6-verification-order-and-error-codes)
7. [Worked Example](#7-worked-example)
8. [Protected Endpoints](#8-protected-endpoints)
9. [Known Limitations of v1](#9-known-limitations-of-v1)
10. [Contract Event Schema Reference](#10-contract-event-schema-reference)
    - [Encoding conventions](#encoding-conventions)
    - [SDK event parser](#sdk-event-parser)
    - [Core contract events](#core-contract-events)
    - [Companion contract events](#companion-contract-events)
    - [Inactive source modules and absent events](#inactive-source-modules-and-absent-events)

---

## 1. Stellar HTTP Authentication (v1)

Protected endpoints authenticate the caller by an Ed25519 signature over a message derived
from the request itself. The signature commits to the HTTP method, the request path, and a
hash of the request body, so a credential captured on one endpoint cannot be replayed
against another.

The canonical implementation lives in `packages/types/src/auth.ts` (`buildAuthMessage`,
`canonicalizeAuthPath`) and is shared by the verifier and the first-party clients. External
integrators should implement the format below directly — see the
[migration guide](./MIGRATION_AUTH_V1.md) for a self-contained reference implementation.

### Why the `v1:` prefix

The message begins with a version tag so the scheme can be rotated without a flag day. A
future `v2:` can add fields (a nonce, header binding, the query string) while a server
accepts both prefixes during a transition window, routing each credential to the verifier
that matches its tag. Without the prefix, any change to the layout would require every
client and every server to cut over in the same instant.

The prefix is part of the signed bytes, so it cannot be swapped by an attacker to downgrade
a v2 credential into a v1 one.

---

## 2. The Signed Message

```
v1:{METHOD}:{canonicalPath}:{address}:{timestamp}:{bodyHash}
```

| Field           | Description                                                                |
| --------------- | -------------------------------------------------------------------------- |
| `v1`            | Literal version tag.                                                       |
| `METHOD`        | HTTP method, **upper-cased**: `GET`, `POST`, `PATCH`, `DELETE`.            |
| `canonicalPath` | Request path after canonicalisation — see [§3](#3-canonical-path).         |
| `address`       | Signer's Stellar public key (`G…`), exactly as sent in the header payload. |
| `timestamp`     | Unix epoch in **milliseconds**, as an integer with no separators.          |
| `bodyHash`      | Lowercase hex SHA-256 of the raw body — see [§4](#4-body-hash).            |

Fields are joined with a literal `:`. No field is escaped or length-prefixed. A path _can_
legitimately contain a `:` (`/api/items:batchGet` is a valid URL), so the message is not
unambiguous by construction — it is unambiguous because the last three fields have fixed
shapes (a 56-character base32 address, digits, 64 hex characters), which lets the suffix be
read unambiguously from the right no matter what the path contains.

What gets signed is the **SHA-256 digest of this message**, not the message itself:

```
signature = Ed25519_sign(privateKey, SHA256(message))
```

The digest is signed as 32 raw bytes. The signature is the raw 64-byte Ed25519 output,
base64-encoded for transport.

> **Wallet signers do not do this today.** The rule above is what the server verifies, and it
> is what a signer holding the key directly (`Keypair.sign`) produces. Browser wallets are a
> different matter: Freighter's `signBlob` treats its argument as an opaque **message**, wraps
> it in the SEP-0053 envelope and signs `SHA256("Stellar Signed Message:\n" + blob)` — never
> the 32 raw bytes. It also returns a `Buffer`, not the base64 string its TypeScript signature
> promises. Both facts are verified against a real Freighter 2.0.0 signature. Consequently the
> first-party browser client **does not currently interoperate with this server**, and
> reconciling the two is an open decision, not something this document describes as settled.
> If you sign with a wallet, expect the envelope; if you sign with a library, follow the rule
> above.

---

## 3. Canonical Path

The server canonicalises `req.originalUrl`, which always carries the **full path including
any router mount prefix**. Clients must sign the same absolute path.

The rule, in order:

1. Drop everything from the first `?` onward.
2. Strip all trailing `/`.
3. If the result is empty, use `/`.

| Raw path                 | Canonical path  |
| ------------------------ | --------------- |
| `/api/follows`           | `/api/follows`  |
| `/api/follows/`          | `/api/follows`  |
| `/api/follows//`         | `/api/follows`  |
| `/api/follows?cursor=5`  | `/api/follows`  |
| `/api/follows/?cursor=5` | `/api/follows`  |
| `/`                      | `/`             |
| `//`                     | `/`             |
| `/api//follows`          | `/api//follows` |

Two consequences worth internalising:

- Sign `/api/notifications/preferences`, never the router-relative `/preferences`. Signing
  the relative path produces a different message and the request comes back `401`.
- If your base URL carries a path segment of its own (`https://example.com/indexer`), that
  segment reaches the server as part of `req.originalUrl` but never appears in the path you
  passed to the signer. Keep the base URL an origin only, or include the prefix in the
  signed path.

---

## 4. Body Hash

`bodyHash` is the lowercase hex SHA-256 of the **exact bytes** sent as the request body.

When there is no body, hash the empty string:

```
SHA256("") = e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

This is the value used for every `GET`, and for any request sent without a body. The server
reaches the same value either way: a request with no JSON content type never triggers
body-parser's `verify` hook and its absent raw body is hashed as empty, while a JSON request
with an empty body triggers the hook with a zero-length buffer, which hashes to the same
digest.

> **Hash the bytes you send, not the object you serialised.** Serialise once into a
> variable, then use that same variable for both the hash and the request body. Serialising
> separately for each can yield different bytes — key order, whitespace, number
> formatting — and the signature would then commit to a body that was never sent, producing
> a `401` that is very hard to read.

---

## 5. The Authorization Header

```
Authorization: StellarSig <base64(JSON)>
```

The base64 payload decodes to a JSON object with exactly these three fields:

```json
{
  "address": "G...",
  "timestamp": 1735689600000,
  "signature": "<base64 of the raw 64-byte Ed25519 signature>"
}
```

The header value is the literal `StellarSig`, one space, then the base64 payload. The parser
splits on a single space and rejects anything else, so the payload must not contain spaces.

`address` and `timestamp` are sent in the clear **and** covered by the signature — the
server uses the transmitted values to rebuild the message, then checks the signature against
it. Tampering with either produces a message that no longer verifies.

---

## 6. Verification Order and Error Codes

The server checks in this order and returns on the first failure:

| Order | Check                                 | Status | Code                  |
| ----- | ------------------------------------- | ------ | --------------------- |
| 1     | Header present, parsable, all fields  | `400`  | `INVALID_AUTH_HEADER` |
| 2     | `timestamp` is not in the future      | `403`  | `INVALID_TIMESTAMP`   |
| 3     | `timestamp` is at most 30 s old       | `403`  | `EXPIRED_TIMESTAMP`   |
| 4     | Signature matches the rebuilt message | `401`  | `INVALID_SIGNATURE`   |

Errors carry the shape:

```json
{
  "error": {
    "code": "INVALID_SIGNATURE",
    "message": "Invalid signature",
    "requestId": "..."
  }
}
```

**The order matters when diagnosing a replay.** Timestamp checks run before the signature
check, so an expired credential returns `403` regardless of which path it is presented at —
you will not see the `401` that a wrong path would otherwise produce. A credential replayed
against a different path _within_ the tolerance window is what yields `401`.

The tolerance is 30 000 ms, defined by `SIGNATURE_TIMESTAMP_TOLERANCE_MS` in
`services/indexer/src/middleware/stellarAuth.ts`. Clock skew between client and server eats
directly into this budget; a client running more than 30 s fast is rejected outright as
`INVALID_TIMESTAMP`.

---

## 7. Worked Example

Every value below is reproducible — the keypair is seeded with 32 bytes of `0x07`.

```
address     GDVEU3DD4KOFECV66VIHWEZOYX4ZKR3WV27L464SIIPOU2IUI3JCZA57
method      POST
path        /api/follows
timestamp   1735689600000
body        {"followee":"GBRPYHIL2CI3FNQ4BXLFMNDLFJUNPU2HY3ZMFSHONUCEOASW7QC7OX2H"}
```

Step 1 — hash the body:

```
bodyHash = 420fe3b7b804bb866cbfeb113d913ba1543099424aed7694297cc3f042ee29a9
```

Step 2 — build the message:

```
v1:POST:/api/follows:GDVEU3DD4KOFECV66VIHWEZOYX4ZKR3WV27L464SIIPOU2IUI3JCZA57:1735689600000:420fe3b7b804bb866cbfeb113d913ba1543099424aed7694297cc3f042ee29a9
```

Step 3 — digest it (base64, for inspection):

```
1bWsu8SkJG10Q5vEoWQZdo3ZIbseNrILzJmuOsP2VyM=
```

Step 4 — sign the digest and base64 the signature:

```
evOK1CFjMZSTvHtuXrxhDVImMQi9JByqzurli1avrRrEPyjvhhQHYwyAmJLvicSAgHEnOte0qXFBlafhOHjtAw==
```

Step 5 — assemble the header:

```
Authorization: StellarSig eyJhZGRyZXNzIjoiR0RWRVUzREQ0S09GRUNWNjZWSUhXRVpPWVg0WktSM1dWMjdMNDY0U0lJUE9VMklVSTNKQ1pBNTciLCJ0aW1lc3RhbXAiOjE3MzU2ODk2MDAwMDAsInNpZ25hdHVyZSI6ImV2T0sxQ0ZqTVpTVHZIdHVYcnhoRFZJbU1RaTlKQnlxenVybGkxYXZyUnJFUHlqdmhoUUhZd3lBbUpMdmljU0FnSEVuT3RlMHFYRkJsYWZoT0hqdEF3PT0ifQ==
```

Note that this example's timestamp is long past, so replaying it against a live server
returns `403 EXPIRED_TIMESTAMP`. It is for verifying your message construction, not for
testing a live endpoint.

---

## 8. Protected Endpoints

| Method | Path                             | Body |
| ------ | -------------------------------- | ---- |
| `POST` | `/api/notifications/register`    | JSON |
| `POST` | `/api/notifications/deregister`  | JSON |
| `GET`  | `/api/notifications/preferences` | none |
| `POST` | `/api/notifications/preferences` | JSON |
| `POST` | `/api/messages`                  | JSON |

All other indexer endpoints are public reads and take no `Authorization` header.

---

## 9. Known Limitations of v1

The signature covers the method, the canonical path, and the body. Everything else about the
request — headers, query string, transport — is outside it. Three consequences deserve to be
stated plainly.

### 9.1 The query string is not signed

Canonicalisation strips everything after `?` before signing, so query parameters are **not**
authenticated. A credential captured for `/api/posts?limit=10` is equally valid for
`/api/posts?limit=1000` within the tolerance window.

No currently protected endpoint reads query parameters, so there is nothing to tamper with
today. That is a property of the current route set, not a guarantee of the scheme — **do not
add a query parameter to a protected endpoint** without moving it into the body or extending
the scheme to `v2`.

### 9.2 There is no nonce — identical replay stays possible

The scheme binds a credential to _a_ request shape, not to _one_ delivery of it. Anyone who
observes a request can resend it byte-for-byte — same method, same path, same body — and it
will be accepted until the 30-second window closes. For a non-idempotent endpoint that means
the action happens twice.

The window is the only bound. Defending against identical replay needs a nonce carried in
the signed message plus server-side storage of spent nonces for at least the tolerance
period, which v1 does not have. Until then, TLS is what stands between an observer and a
replayable credential: **these endpoints must not be served over plaintext HTTP.**

### 9.3 A non-JSON body would go unsigned

The server captures the raw body through `express.json({ verify })`, and body-parser only
invokes that hook when the request's `Content-Type` is JSON. For any other content type the
raw body is never captured, `bodyHash` falls back to the hash of the empty string, and the
body travels **entirely outside the signature** — a `multipart/form-data` upload on a
protected route would be freely tamperable while the request still verifies.

Every protected endpoint is JSON-only today, so this is latent rather than live. It becomes a
live vulnerability the moment a protected route accepts another content type. **Any new
protected endpoint must be JSON**, or the raw-body capture in
`services/indexer/src/middleware/rawBody.ts` must be widened to cover its content type first.

---

## 10. Contract Event Schema Reference

This catalog covers all 50 event declarations in the active core contract source and the nine
events in the token-factory, referral, vesting, and multicall contracts. Three declarations
in currently inactive core source modules are listed separately. The Rust declarations
linked below define the wire schema; the SDK's legacy normalized types are a separate layer.

### Encoding conventions

Under the locked Soroban SDK 26.1.0, `#[contractevent]` adds the struct name in
`snake_case` as the **first topic**, including the `_event` suffix when present.
An explicit `topics = [...]` replaces that prefix. Fields marked `#[topic]` follow in
declaration order; all remaining fields form a **map** keyed by field-name symbols.
An event without data fields has an empty map, not a null payload. See the
[SDK macro definition](https://github.com/stellar/rs-soroban-sdk/blob/v26.1.0/soroban-sdk/src/lib.rs).

The examples are **decoded documentation JSON**, not RPC-ready payloads:

| Rust value                            | Representation in these examples                        | Native decoding / handling                                                                                           |
| ------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `Address`                             | Valid example `G...` account or `C...` contract address | Address string. Example accounts are fixtures, not deployment recipients.                                            |
| `Symbol`, `String`                    | JSON string                                             | Preserve case and spelling.                                                                                          |
| `u32`, `bool`                         | JSON number / boolean                                   | JavaScript `number` / `boolean`.                                                                                     |
| `u64`, `i128`                         | Decimal **string**                                      | Normally `bigint`; stringify for JSON to avoid precision loss. Token amounts are base units, not formatted balances. |
| `BytesN<32>`                          | 64-character lowercase **hex** string                   | Native `Uint8Array`; convert hex explicitly when constructing XDR. This is not base64.                               |
| `Vec<Address>`                        | JSON array of addresses                                 | Native array.                                                                                                        |
| `Role`, `PauseModule`, `GovParameter` | Tagged array, e.g. `["Admin"]`                          | Unit variants of `#[contracttype]` enums encode as a vector containing the variant symbol.                           |

Every `version` topic is `EVENT_VERSION = 1`. `ContractUpgraded` and `PostDeleted`
do not declare a version. Ledger numbers are ledger sequences; `reveal_time`,
`window_start`, `window_end`, `voting_ends_at`, and vesting times are Unix seconds
(vesting cliff/duration fields are elapsed seconds).

RPC `topic[]` / `value` (or adapter `topics[]` / `data`) contain base64-encoded `ScVal`
XDR for each topic and for the data map. The `type` in the RPC envelope is the event
category (such as `contract`), not the Rust event name or SDK discriminator. Contract ID,
ledger, transaction hash, event ID, and paging token are envelope metadata, omitted from
each decoded example. Filter by emitting contract ID as well as topic; token transfers
performed by these contracts can also emit external token-contract events.

### SDK event parser

- [`events/types.ts`](../packages/sdk/src/events/types.ts) defines `SorobanEvent`,
  `LinkoraEvent`, `EVENT_NAMES`, and `parseContractEvent`.
- [`events/parser.ts`](../packages/sdk/src/events/parser.ts) exposes `parseEvent(raw,
{ tokenFactoryId })`, delegating to `parseContractEvent`. The token factory ID changes
  `meta.source.kind`; it does not add factory event decoding support.
- [`events/subscriber.ts`](../packages/sdk/src/events/subscriber.ts) dispatches normalized
  event types to subscribed handlers. Preserve the raw RPC event and cursor for replay
  when an event cannot be parsed.
- [`generated/events.ts`](../packages/sdk/src/generated/events.ts) is a hand-maintained
  legacy wrapper, not the contract schema source. ABI bindings live under
  [`generated/contracts`](../packages/sdk/src/generated/contracts).

**Current compatibility limits:** `EVENT_NAMES` recognizes Rust-name/legacy aliases,
such as `FollowEvent` and `follow`, but generally does not recognize the actual default
prefix `follow_event`. Also, `payloadFrom()` merges data maps and **object-valued**
topics; it does not associate positional scalar topics with field names. The data-only
`contract_upgraded` event can decode, but recognized prefixes such as `post_deleted` and
the explicit `withdrawal_slippage_exceeded` still lack positional topic decoding. Numeric missing
fields may make the parser return `null`; string conversions may produce `"undefined"`.
Unknown prefixes and decoding exceptions return `null`.

The **SDK alias** recorded per event below means an existing normalized discriminator,
not a claim of raw wire compatibility. “Not in `LinkoraEvent`” means that the union and
parser switch do not cover that declaration. To consume these schemas today, decode the
positional topics and map yourself using the Rust schema, or normalize all named fields
and use a recognized alias before passing an event to the legacy parser. Byte fields in
that adapter must retain their native types; the documentation hex strings are not input.
The parser returns credential roots/nullifiers and attestation report hashes as base64
strings, while public keys/content hashes/Wasm hashes remain `Uint8Array`.

For example, a decoded `FollowEvent` on the wire has
`["follow_event", 1, follower, followee]` and `{}` data. The existing parser instead
accepts an adapter-created event with an encoded `"follow"` topic and a data map holding
`{ follower, followee }`. These are different representations. Do not treat a `null`
result as proof that no contract event occurred.

Changing a field's type, position, prefix, or topic/data placement changes the wire
contract. Version and release such changes together with consumers; keep old decoders
available for replay. Ordinary reverted invocations do not commit their events.

### Core contract events

#### RentPaidEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L518).
SDK alias: `rent_paid`.

`extended_to_ledger` is the new expiry ledger, not a wall-clock timestamp.

| Field location | Ordered topics / named data fields                                                       |
| -------------- | ---------------------------------------------------------------------------------------- |
| Topics         | `"rent_paid_event"`, `version: u32`, `user: Address`, `payer: Address`, `token: Address` |
| Data map       | `amount: i128`, `extended_to_ledger: u32`                                                |

```json
{
  "topics": [
    "rent_paid_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "CACAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAINCW"
  ],
  "data": {
    "amount": "10000000",
    "extended_to_ledger": 123456
  }
}
```

#### ProfileSetEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L533).
SDK alias: `profile_set`.

| Field location | Ordered topics / named data fields                     |
| -------------- | ------------------------------------------------------ |
| Topics         | `"profile_set_event"`, `version: u32`, `user: Address` |
| Data map       | `username: String`                                     |

```json
{
  "topics": ["profile_set_event", 1, "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"],
  "data": {
    "username": "alice"
  }
}
```

#### FollowEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L543).
SDK alias: `follow`.

| Field location | Ordered topics / named data fields                                         |
| -------------- | -------------------------------------------------------------------------- |
| Topics         | `"follow_event"`, `version: u32`, `follower: Address`, `followee: Address` |
| Data map       | Empty map (`{}`)                                                           |

```json
{
  "topics": [
    "follow_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U"
  ],
  "data": {}
}
```

#### UnfollowEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L554).
SDK alias: `unfollow`.

| Field location | Ordered topics / named data fields                                           |
| -------------- | ---------------------------------------------------------------------------- |
| Topics         | `"unfollow_event"`, `version: u32`, `follower: Address`, `followee: Address` |
| Data map       | Empty map (`{}`)                                                             |

```json
{
  "topics": [
    "unfollow_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U"
  ],
  "data": {}
}
```

#### BlockEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L565).
SDK alias: `block`.

| Field location | Ordered topics / named data fields                                      |
| -------------- | ----------------------------------------------------------------------- |
| Topics         | `"block_event"`, `version: u32`, `blocker: Address`, `blocked: Address` |
| Data map       | Empty map (`{}`)                                                        |

```json
{
  "topics": [
    "block_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U"
  ],
  "data": {}
}
```

#### UnblockEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L576).
SDK alias: `unblock`.

| Field location | Ordered topics / named data fields                                        |
| -------------- | ------------------------------------------------------------------------- |
| Topics         | `"unblock_event"`, `version: u32`, `blocker: Address`, `blocked: Address` |
| Data map       | Empty map (`{}`)                                                          |

```json
{
  "topics": [
    "unblock_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U"
  ],
  "data": {}
}
```

#### PostCreatedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L587).
SDK alias: `post_created`.

| Field location | Ordered topics / named data fields                                   |
| -------------- | -------------------------------------------------------------------- |
| Topics         | `"post_created_event"`, `version: u32`, `id: u64`, `author: Address` |
| Data map       | Empty map (`{}`)                                                     |

```json
{
  "topics": [
    "post_created_event",
    1,
    "1",
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {}
}
```

#### PostCommittedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L600).
SDK alias: `post_committed`.

| Field location | Ordered topics / named data fields                                          |
| -------------- | --------------------------------------------------------------------------- |
| Topics         | `"post_committed_event"`, `version: u32`, `post_id: u64`, `author: Address` |
| Data map       | `content_hash: BytesN<32>`, `reveal_time: u64`                              |

```json
{
  "topics": [
    "post_committed_event",
    1,
    "1",
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {
    "content_hash": "0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a",
    "reveal_time": "1800003600"
  }
}
```

#### PostRevealedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L615).
SDK alias: `post_revealed`.

| Field location | Ordered topics / named data fields                                         |
| -------------- | -------------------------------------------------------------------------- |
| Topics         | `"post_revealed_event"`, `version: u32`, `post_id: u64`, `author: Address` |
| Data map       | `content_hash: BytesN<32>`                                                 |

```json
{
  "topics": [
    "post_revealed_event",
    1,
    "1",
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {
    "content_hash": "0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a"
  }
}
```

#### TipEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L627).
SDK alias: `tip`.

| Field location | Ordered topics / named data fields                               |
| -------------- | ---------------------------------------------------------------- |
| Topics         | `"tip_event"`, `version: u32`, `tipper: Address`, `post_id: u64` |
| Data map       | `amount: i128`, `fee: i128`                                      |

```json
{
  "topics": ["tip_event", 1, "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR", "1"],
  "data": {
    "amount": "10000000",
    "fee": "250000"
  }
}
```

#### PoolDepositEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L640).
SDK alias: `pool_deposit`.

| Field location | Ordered topics / named data fields                                              |
| -------------- | ------------------------------------------------------------------------------- |
| Topics         | `"pool_deposit_event"`, `version: u32`, `depositor: Address`, `pool_id: Symbol` |
| Data map       | `amount: i128`                                                                  |

```json
{
  "topics": [
    "pool_deposit_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "community"
  ],
  "data": {
    "amount": "10000000"
  }
}
```

#### PoolWithdrawEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L652).
SDK alias: `pool_withdraw`.

| Field location | Ordered topics / named data fields                                               |
| -------------- | -------------------------------------------------------------------------------- |
| Topics         | `"pool_withdraw_event"`, `version: u32`, `recipient: Address`, `pool_id: Symbol` |
| Data map       | `amount: i128`                                                                   |

```json
{
  "topics": [
    "pool_withdraw_event",
    1,
    "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U",
    "community"
  ],
  "data": {
    "amount": "10000000"
  }
}
```

#### WithdrawalSlippageExceededEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L668).
SDK alias: `withdrawal_slippage_exceeded`.

Diagnostic only: the withdrawal reverts after publishing this event, so it is rolled back and never appears in committed ledger events.

| Field location | Ordered topics / named data fields                                        |
| -------------- | ------------------------------------------------------------------------- |
| Topics         | `"withdrawal_slippage_exceeded"`, `version: u32`, `pool_id: Symbol`       |
| Data map       | `requested_amount: i128`, `received_amount: i128`, `min_out_amount: i128` |

```json
{
  "topics": ["withdrawal_slippage_exceeded", 1, "community"],
  "data": {
    "requested_amount": "10000000",
    "received_amount": "9000000",
    "min_out_amount": "9500000"
  }
}
```

#### PoolCreatedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L680).
SDK alias: `pool_created`.

| Field location | Ordered topics / named data fields                         |
| -------------- | ---------------------------------------------------------- |
| Topics         | `"pool_created_event"`, `version: u32`, `pool_id: Symbol`  |
| Data map       | `token: Address`, `admins: Vec<Address>`, `threshold: u32` |

```json
{
  "topics": ["pool_created_event", 1, "community"],
  "data": {
    "token": "CACAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAINCW",
    "admins": [
      "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
      "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U"
    ],
    "threshold": 2
  }
}
```

#### LikePostEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L692).
SDK alias: `like`.

| Field location | Ordered topics / named data fields                                   |
| -------------- | -------------------------------------------------------------------- |
| Topics         | `"like_post_event"`, `version: u32`, `user: Address`, `post_id: u64` |
| Data map       | Empty map (`{}`)                                                     |

```json
{
  "topics": ["like_post_event", 1, "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR", "1"],
  "data": {}
}
```

#### ContractUpgraded

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L703).
SDK alias: `contract_upgraded`.

| Field location | Ordered topics / named data fields |
| -------------- | ---------------------------------- |
| Topics         | `"contract_upgraded"`              |
| Data map       | `new_wasm_hash: BytesN<32>`        |

```json
{
  "topics": ["contract_upgraded"],
  "data": {
    "new_wasm_hash": "0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a"
  }
}
```

#### PausedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L709).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                 |
| -------------- | -------------------------------------------------- |
| Topics         | `"paused_event"`, `version: u32`, `admin: Address` |
| Data map       | Empty map (`{}`)                                   |

```json
{
  "topics": ["paused_event", 1, "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"],
  "data": {}
}
```

#### UnpausedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L718).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                   |
| -------------- | ---------------------------------------------------- |
| Topics         | `"unpaused_event"`, `version: u32`, `admin: Address` |
| Data map       | Empty map (`{}`)                                     |

```json
{
  "topics": ["unpaused_event", 1, "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"],
  "data": {}
}
```

#### ModulePausedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L727).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields      |
| -------------- | --------------------------------------- |
| Topics         | `"module_paused_event"`, `version: u32` |
| Data map       | `module: PauseModule`, `admin: Address` |

```json
{
  "topics": ["module_paused_event", 1],
  "data": {
    "module": ["Posts"],
    "admin": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  }
}
```

#### ModuleUnpausedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L736).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields        |
| -------------- | ----------------------------------------- |
| Topics         | `"module_unpaused_event"`, `version: u32` |
| Data map       | `module: PauseModule`, `admin: Address`   |

```json
{
  "topics": ["module_unpaused_event", 1],
  "data": {
    "module": ["Posts"],
    "admin": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  }
}
```

#### PostDeleted

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L745).
SDK alias: `post_deleted`.

| Field location | Ordered topics / named data fields                  |
| -------------- | --------------------------------------------------- |
| Topics         | `"post_deleted"`, `post_id: u64`, `author: Address` |
| Data map       | Empty map (`{}`)                                    |

```json
{
  "topics": ["post_deleted", "1", "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"],
  "data": {}
}
```

#### ProfileDeletedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L754).
Not in `LinkoraEvent`.

Reports the deleted username as well as the user; it is distinct from `PostDeleted`.

| Field location | Ordered topics / named data fields                         |
| -------------- | ---------------------------------------------------------- |
| Topics         | `"profile_deleted_event"`, `version: u32`, `user: Address` |
| Data map       | `username: String`                                         |

```json
{
  "topics": [
    "profile_deleted_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {
    "username": "alice"
  }
}
```

#### ProposalCreatedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L764).
SDK alias: `proposal_created`.

| Field location | Ordered topics / named data fields                                                |
| -------------- | --------------------------------------------------------------------------------- |
| Topics         | `"proposal_created_event"`, `version: u32`, `pool_id: Symbol`, `proposal_id: u64` |
| Data map       | `proposer: Address`, `amount: i128`, `recipient: Address`                         |

```json
{
  "topics": ["proposal_created_event", 1, "community", "1"],
  "data": {
    "proposer": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "amount": "10000000",
    "recipient": "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U"
  }
}
```

#### ProposalSignedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L778).
SDK alias: `proposal_signed`.

| Field location | Ordered topics / named data fields                                               |
| -------------- | -------------------------------------------------------------------------------- |
| Topics         | `"proposal_signed_event"`, `version: u32`, `pool_id: Symbol`, `proposal_id: u64` |
| Data map       | `signer: Address`                                                                |

```json
{
  "topics": ["proposal_signed_event", 1, "community", "1"],
  "data": {
    "signer": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  }
}
```

#### ProposalExecutedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L790).
SDK alias: `proposal_executed`.

| Field location | Ordered topics / named data fields                                                 |
| -------------- | ---------------------------------------------------------------------------------- |
| Topics         | `"proposal_executed_event"`, `version: u32`, `pool_id: Symbol`, `proposal_id: u64` |
| Data map       | `amount: i128`, `recipient: Address`                                               |

```json
{
  "topics": ["proposal_executed_event", 1, "community", "1"],
  "data": {
    "amount": "10000000",
    "recipient": "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U"
  }
}
```

#### PoolAdminAddedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L803).
SDK alias: `pool_admin_added`.

| Field location | Ordered topics / named data fields                            |
| -------------- | ------------------------------------------------------------- |
| Topics         | `"pool_admin_added_event"`, `version: u32`, `pool_id: Symbol` |
| Data map       | `new_admin: Address`                                          |

```json
{
  "topics": ["pool_admin_added_event", 1, "community"],
  "data": {
    "new_admin": "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U"
  }
}
```

#### PoolAdminRemovedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L813).
SDK alias: `pool_admin_removed`.

| Field location | Ordered topics / named data fields                              |
| -------------- | --------------------------------------------------------------- |
| Topics         | `"pool_admin_removed_event"`, `version: u32`, `pool_id: Symbol` |
| Data map       | `admin: Address`                                                |

```json
{
  "topics": ["pool_admin_removed_event", 1, "community"],
  "data": {
    "admin": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  }
}
```

#### PoolThresholdUpdatedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L823).
SDK alias: `pool_threshold_updated`.

| Field location | Ordered topics / named data fields                                  |
| -------------- | ------------------------------------------------------------------- |
| Topics         | `"pool_threshold_updated_event"`, `version: u32`, `pool_id: Symbol` |
| Data map       | `old_threshold: u32`, `new_threshold: u32`                          |

```json
{
  "topics": ["pool_threshold_updated_event", 1, "community"],
  "data": {
    "old_threshold": 1,
    "new_threshold": 2
  }
}
```

#### DmKeyPublishedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L834).
SDK alias: `dm_key_published`.

Contains only the public X25519 messaging key; private key material is never part of this event.

| Field location | Ordered topics / named data fields                          |
| -------------- | ----------------------------------------------------------- |
| Topics         | `"dm_key_published_event"`, `version: u32`, `user: Address` |
| Data map       | `public_key: BytesN<32>`                                    |

```json
{
  "topics": [
    "dm_key_published_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {
    "public_key": "0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a"
  }
}
```

#### CredentialRootUpdatedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L844).
SDK alias: `credential_root_updated`.

| Field location | Ordered topics / named data fields                                 |
| -------------- | ------------------------------------------------------------------ |
| Topics         | `"credential_root_updated_event"`, `version: u32`, `user: Address` |
| Data map       | `root: BytesN<32>`                                                 |

```json
{
  "topics": [
    "credential_root_updated_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {
    "root": "0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a"
  }
}
```

#### CredentialVerifiedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L854).
SDK alias: `credential_verified`.

| Field location | Ordered topics / named data fields                                                      |
| -------------- | --------------------------------------------------------------------------------------- |
| Topics         | `"credential_verified_event"`, `version: u32`, `user: Address`, `nullifier: BytesN<32>` |
| Data map       | Empty map (`{}`)                                                                        |

```json
{
  "topics": [
    "credential_verified_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a"
  ],
  "data": {}
}
```

#### FeeUpdatedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L865).
SDK alias: `fee_updated`.

| Field location | Ordered topics / named data fields                    |
| -------------- | ----------------------------------------------------- |
| Topics         | `"fee_updated_event"`, `version: u32`, `name: Symbol` |
| Data map       | `old_fee_bps: u32`, `new_fee_bps: u32`                |

```json
{
  "topics": ["fee_updated_event", 1, "fee_upd"],
  "data": {
    "old_fee_bps": 1,
    "new_fee_bps": 250
  }
}
```

#### TreasuryUpdatedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L876).
SDK alias: `treasury_updated`.

| Field location | Ordered topics / named data fields                         |
| -------------- | ---------------------------------------------------------- |
| Topics         | `"treasury_updated_event"`, `version: u32`, `name: Symbol` |
| Data map       | `old_treasury: Address`, `new_treasury: Address`           |

```json
{
  "topics": ["treasury_updated_event", 1, "treas_upd"],
  "data": {
    "old_treasury": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "new_treasury": "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U"
  }
}
```

#### RoleGrantedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L887).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                       |
| -------------- | ------------------------------------------------------------------------ |
| Topics         | `"role_granted_event"`, `version: u32`, `role: Role`, `account: Address` |
| Data map       | `sender: Address`                                                        |

```json
{
  "topics": [
    "role_granted_event",
    1,
    ["Admin"],
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {
    "sender": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  }
}
```

#### RoleRevokedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L899).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                       |
| -------------- | ------------------------------------------------------------------------ |
| Topics         | `"role_revoked_event"`, `version: u32`, `role: Role`, `account: Address` |
| Data map       | `sender: Address`                                                        |

```json
{
  "topics": [
    "role_revoked_event",
    1,
    ["Admin"],
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {
    "sender": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  }
}
```

#### GovProposalCreatedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L911).
SDK alias: `gov_proposal_created`.

| Field location | Ordered topics / named data fields                                 |
| -------------- | ------------------------------------------------------------------ |
| Topics         | `"gov_proposal_created_event"`, `version: u32`, `proposal_id: u64` |
| Data map       | `proposer: Address`, `parameter: GovParameter`, `new_value: u64`   |

```json
{
  "topics": ["gov_proposal_created_event", 1, "1"],
  "data": {
    "proposer": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "parameter": ["FeeBps"],
    "new_value": "1"
  }
}
```

#### GovVoteEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L923).
SDK alias: `gov_vote`.

| Field location | Ordered topics / named data fields                                       |
| -------------- | ------------------------------------------------------------------------ |
| Topics         | `"gov_vote_event"`, `version: u32`, `proposal_id: u64`, `voter: Address` |
| Data map       | `support: bool`                                                          |

```json
{
  "topics": ["gov_vote_event", 1, "1", "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"],
  "data": {
    "support": true
  }
}
```

#### GovProposalExecutedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L935).
SDK alias: `gov_proposal_executed`.

| Field location | Ordered topics / named data fields                                  |
| -------------- | ------------------------------------------------------------------- |
| Topics         | `"gov_proposal_executed_event"`, `version: u32`, `proposal_id: u64` |
| Data map       | `parameter: GovParameter`, `new_value: u64`                         |

```json
{
  "topics": ["gov_proposal_executed_event", 1, "1"],
  "data": {
    "parameter": ["FeeBps"],
    "new_value": "1"
  }
}
```

#### GovProposalVetoedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L946).
SDK alias: `gov_proposal_vetoed`.

| Field location | Ordered topics / named data fields                                |
| -------------- | ----------------------------------------------------------------- |
| Topics         | `"gov_proposal_vetoed_event"`, `version: u32`, `proposal_id: u64` |
| Data map       | Empty map (`{}`)                                                  |

```json
{
  "topics": ["gov_proposal_vetoed_event", 1, "1"],
  "data": {}
}
```

#### EmergencyBypassEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L955).
SDK alias: `emergency_bypass`.

Identifies an immediate administrative bypass action, such as `set_fee` or `set_tres` (the `set_treasury` entry point); the related parameter update event is also emitted.

| Field location | Ordered topics / named data fields                           |
| -------------- | ------------------------------------------------------------ |
| Topics         | `"emergency_bypass_event"`, `version: u32`, `action: Symbol` |
| Data map       | Empty map (`{}`)                                             |

```json
{
  "topics": ["emergency_bypass_event", 1, "set_fee"],
  "data": {}
}
```

#### AttestationVerifiedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L964).
SDK alias: `attestation_verified`.

| Field location | Ordered topics / named data fields                                                               |
| -------------- | ------------------------------------------------------------------------------------------------ |
| Topics         | `"attestation_verified_event"`, `version: u32`, `oracle_name: Symbol`, `report_hash: BytesN<32>` |
| Data map       | `creator: Address`, `window_start: u64`, `window_end: u64`                                       |

```json
{
  "topics": [
    "attestation_verified_event",
    1,
    "analytics",
    "0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a"
  ],
  "data": {
    "creator": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "window_start": "1800000000",
    "window_end": "1800003600"
  }
}
```

#### PostReportedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L978).
SDK alias: `post_reported`.

| Field location | Ordered topics / named data fields                                           |
| -------------- | ---------------------------------------------------------------------------- |
| Topics         | `"post_reported_event"`, `version: u32`, `post_id: u64`, `reporter: Address` |
| Data map       | `stake_amount: i128`                                                         |

```json
{
  "topics": [
    "post_reported_event",
    1,
    "1",
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {
    "stake_amount": "1"
  }
}
```

#### PostRemovedByModerationEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L990).
SDK alias: `post_removed_by_moderation`.

| Field location | Ordered topics / named data fields                                                        |
| -------------- | ----------------------------------------------------------------------------------------- |
| Topics         | `"post_removed_by_moderation_event"`, `version: u32`, `post_id: u64`, `reporter: Address` |
| Data map       | Empty map (`{}`)                                                                          |

```json
{
  "topics": [
    "post_removed_by_moderation_event",
    1,
    "1",
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {}
}
```

#### ReportDismissedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L1001).
SDK alias: `report_dismissed`.

| Field location | Ordered topics / named data fields                                              |
| -------------- | ------------------------------------------------------------------------------- |
| Topics         | `"report_dismissed_event"`, `version: u32`, `post_id: u64`, `reporter: Address` |
| Data map       | Empty map (`{}`)                                                                |

```json
{
  "topics": [
    "report_dismissed_event",
    1,
    "1",
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {}
}
```

#### PostContentHashSetEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L1012).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                                 |
| -------------- | ---------------------------------------------------------------------------------- |
| Topics         | `"post_content_hash_set_event"`, `version: u32`, `post_id: u64`, `author: Address` |
| Data map       | `content_hash: BytesN<32>`                                                         |

```json
{
  "topics": [
    "post_content_hash_set_event",
    1,
    "1",
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {
    "content_hash": "0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a"
  }
}
```

#### AppealFiledEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L1024).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                           |
| -------------- | ---------------------------------------------------------------------------- |
| Topics         | `"appeal_filed_event"`, `version: u32`, `post_id: u64`, `appellant: Address` |
| Data map       | Empty map (`{}`)                                                             |

```json
{
  "topics": [
    "appeal_filed_event",
    1,
    "1",
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {}
}
```

#### AppealResolvedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L1035).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                          |
| -------------- | --------------------------------------------------------------------------- |
| Topics         | `"appeal_resolved_event"`, `version: u32`, `post_id: u64`, `approved: bool` |
| Data map       | Empty map (`{}`)                                                            |

```json
{
  "topics": ["appeal_resolved_event", 1, "1", true],
  "data": {}
}
```

#### GovAppealFiledEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L1046).
Not in `LinkoraEvent`.

Token-holder governance appeal; the explicit `appeal_filed` prefix differs from the moderator appeal `appeal_filed_event`.

| Field location | Ordered topics / named data fields                                 |
| -------------- | ------------------------------------------------------------------ |
| Topics         | `"appeal_filed"`, `version: u32`, `appeal_id: u64`, `post_id: u64` |
| Data map       | `appellant: Address`, `stake_amount: i128`, `voting_ends_at: u64`  |

```json
{
  "topics": ["appeal_filed", 1, "1", "1"],
  "data": {
    "appellant": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "stake_amount": "1",
    "voting_ends_at": "1800003600"
  }
}
```

#### GovAppealVotedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L1060).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                   |
| -------------- | -------------------------------------------------------------------- |
| Topics         | `"appeal_voted"`, `version: u32`, `appeal_id: u64`, `voter: Address` |
| Data map       | `support: bool`, `weight: i128`                                      |

```json
{
  "topics": ["appeal_voted", 1, "1", "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"],
  "data": {
    "support": true,
    "weight": "1"
  }
}
```

#### GovAppealResolvedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/lib.rs#L1073).
Not in `LinkoraEvent`.

Includes the voting outcome and both slashing totals; the explicit prefix differs from `appeal_resolved_event`.

| Field location | Ordered topics / named data fields                                                                             |
| -------------- | -------------------------------------------------------------------------------------------------------------- |
| Topics         | `"appeal_resolved"`, `version: u32`, `appeal_id: u64`, `post_id: u64`                                          |
| Data map       | `passed: bool`, `votes_for: i128`, `votes_against: i128`, `moderator_slashed: i128`, `appellant_slashed: i128` |

```json
{
  "topics": ["appeal_resolved", 1, "1", "1"],
  "data": {
    "passed": true,
    "votes_for": "1",
    "votes_against": "1",
    "moderator_slashed": "1",
    "appellant_slashed": "1"
  }
}
```

### Companion contract events

These events are emitted by separate contract IDs, not the core Linkora contract.

**Token factory**

#### CreatorTokenDeployedEvent

[Rust declaration](../packages/contracts/contracts/token-factory/src/lib.rs#L40).
Not in `LinkoraEvent`.

Reports token creation; metadata registry insertion is a separate `CreatorTokenRegisteredEvent`.

| Field location | Ordered topics / named data fields                                                              |
| -------------- | ----------------------------------------------------------------------------------------------- |
| Topics         | `"creator_token_deployed_event"`, `version: u32`, `deployer: Address`, `token_address: Address` |
| Data map       | `name: String`, `symbol: String`                                                                |

```json
{
  "topics": [
    "creator_token_deployed_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "CACAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAINCW"
  ],
  "data": {
    "name": "Creator Token",
    "symbol": "ALICE"
  }
}
```

#### CreatorTokenRegisteredEvent

[Rust declaration](../packages/contracts/contracts/token-factory/src/lib.rs#L59).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                                               |
| -------------- | ------------------------------------------------------------------------------------------------ |
| Topics         | `"creator_token_registered_event"`, `version: u32`, `creator: Address`, `token_address: Address` |
| Data map       | `symbol: String`, `deployed_ledger: u32`                                                         |

```json
{
  "topics": [
    "creator_token_registered_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "CACAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAINCW"
  ],
  "data": {
    "symbol": "ALICE",
    "deployed_ledger": 123456
  }
}
```

**Referral contract**

#### ReferralRegisteredEvent

[Rust declaration](../packages/contracts/contracts/linkora-referral/src/lib.rs#L51).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                             |
| -------------- | ------------------------------------------------------------------------------ |
| Topics         | `"referral_registered_event"`, `version: u32`, `user: Address`, `code: String` |
| Data map       | `referred: Address`                                                            |

```json
{
  "topics": [
    "referral_registered_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "alice-referral"
  ],
  "data": {
    "referred": "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U"
  }
}
```

#### ReferralRewardPaidEvent

[Rust declaration](../packages/contracts/contracts/linkora-referral/src/lib.rs#L68).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                                       |
| -------------- | ---------------------------------------------------------------------------------------- |
| Topics         | `"referral_reward_paid_event"`, `version: u32`, `referrer: Address`, `referred: Address` |
| Data map       | `tip_amount: i128`, `reward_amount: i128`, `rate_bps: u32`                               |

```json
{
  "topics": [
    "referral_reward_paid_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "GCATS5YOVB6ROX2WUNKGNQ2MP3GMXDMKSG2O4N5CLX3A6W4PZGZZI55U"
  ],
  "data": {
    "tip_amount": "10000000",
    "reward_amount": "250000",
    "rate_bps": 250
  }
}
```

#### ReferralRateUpdatedEvent

[Rust declaration](../packages/contracts/contracts/linkora-referral/src/lib.rs#L89).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields              |
| -------------- | ----------------------------------------------- |
| Topics         | `"referral_rate_updated_event"`, `version: u32` |
| Data map       | `old_rate_bps: u32`, `new_rate_bps: u32`        |

```json
{
  "topics": ["referral_rate_updated_event", 1],
  "data": {
    "old_rate_bps": 1,
    "new_rate_bps": 250
  }
}
```

**Vesting contract**

#### VestCreatedEvent

[Rust declaration](../packages/contracts/contracts/linkora-vesting/src/lib.rs#L86).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                                                       |
| -------------- | -------------------------------------------------------------------------------------------------------- |
| Topics         | `"vest_created"`, `version: u32`, `vest_id: u64`, `beneficiary: Address`                                 |
| Data map       | `token: Address`, `total_amount: i128`, `start_time: u64`, `cliff_seconds: u64`, `duration_seconds: u64` |

```json
{
  "topics": ["vest_created", 1, "1", "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"],
  "data": {
    "token": "CACAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAIBAEAQCAINCW",
    "total_amount": "10000000",
    "start_time": "1800000000",
    "cliff_seconds": "86400",
    "duration_seconds": "2592000"
  }
}
```

#### TokensClaimedEvent

[Rust declaration](../packages/contracts/contracts/linkora-vesting/src/lib.rs#L102).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                         |
| -------------- | -------------------------------------------------------------------------- |
| Topics         | `"tokens_claimed"`, `version: u32`, `vest_id: u64`, `beneficiary: Address` |
| Data map       | `amount: i128`, `total_claimed: i128`                                      |

```json
{
  "topics": ["tokens_claimed", 1, "1", "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"],
  "data": {
    "amount": "10000000",
    "total_claimed": "1"
  }
}
```

#### VestRevokedEvent

[Rust declaration](../packages/contracts/contracts/linkora-vesting/src/lib.rs#L115).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                       |
| -------------- | ------------------------------------------------------------------------ |
| Topics         | `"vest_revoked"`, `version: u32`, `vest_id: u64`, `beneficiary: Address` |
| Data map       | `returned_amount: i128`, `vested_amount: i128`                           |

```json
{
  "topics": ["vest_revoked", 1, "1", "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"],
  "data": {
    "returned_amount": "1",
    "vested_amount": "1"
  }
}
```

**Multicall contract**

#### BatchExecutedEvent

[Rust declaration](../packages/contracts/contracts/linkora-multicall/src/lib.rs#L53).
Not in `LinkoraEvent`.

Published once after a successful atomic batch. A failed batch reverts its effects and does not commit this event.

| Field location | Ordered topics / named data fields                    |
| -------------- | ----------------------------------------------------- |
| Topics         | `"batch_executed"`, `version: u32`, `caller: Address` |
| Data map       | `call_count: u32`                                     |

```json
{
  "topics": ["batch_executed", 1, "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"],
  "data": {
    "call_count": 2
  }
}
```

### Inactive source modules and absent events

`upgrade_hardening.rs` and `username_rename.rs` contain the declarations below, but
`linkora-contracts/src/lib.rs` does not declare either module. They are therefore not
compiled into the current contract and do not provide callable upgrade-ceremony or rename
APIs. Their schemas are recorded for source completeness, not as deployed capabilities.

The issue examples `OracleRotatedEvent` and `BatchBumpEvent` do **not** exist in this
revision. `register_oracle` updates the stored oracle key without publishing an event;
`batch_bump_user_graph` returns a bump count without publishing an event. There is no
topic or data schema to document for either proposed event. Adding them requires a
contract change and corresponding parser/versioning work.

**Inactive upgrade module**

#### UpgradeProposedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/upgrade_hardening.rs#L19).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                            |
| -------------- | ----------------------------------------------------------------------------- |
| Topics         | `"upgrade_proposed_event"`, `version: u32`                                    |
| Data map       | `proposer: Address`, `new_wasm_hash: BytesN<32>`, `execute_after_ledger: u32` |

```json
{
  "topics": ["upgrade_proposed_event", 1],
  "data": {
    "proposer": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "new_wasm_hash": "0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a",
    "execute_after_ledger": 123456
  }
}
```

#### UpgradeExecutedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/upgrade_hardening.rs#L29).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                                          |
| -------------- | --------------------------------------------------------------------------- |
| Topics         | `"upgrade_executed_event"`, `version: u32`                                  |
| Data map       | `executor: Address`, `new_wasm_hash: BytesN<32>`, `executed_at_ledger: u32` |

```json
{
  "topics": ["upgrade_executed_event", 1],
  "data": {
    "executor": "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR",
    "new_wasm_hash": "0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a",
    "executed_at_ledger": 123456
  }
}
```

**Inactive username module**

#### UsernameRenamedEvent

[Rust declaration](../packages/contracts/contracts/linkora-contracts/src/username_rename.rs#L19).
Not in `LinkoraEvent`.

| Field location | Ordered topics / named data fields                          |
| -------------- | ----------------------------------------------------------- |
| Topics         | `"username_renamed_event"`, `version: u32`, `user: Address` |
| Data map       | `old_username: String`, `new_username: String`              |

```json
{
  "topics": [
    "username_renamed_event",
    1,
    "GCFIRY65OQE7DFP5KLNS2PF2LVZMUZYJX4OZIEQ36N2IQANUB5XVYOJR"
  ],
  "data": {
    "old_username": "alice",
    "new_username": "alice_new"
  }
}
```
