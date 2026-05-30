# BRC-XX: Single-Use Signed Proofs for Request Authentication

Kjartan Oskarsson (kjartan.oskarsson@bsvassociation.org)

> The BRC number above is a placeholder to be assigned on acceptance.

## Abstract

This standard defines a lightweight mechanism for a server to authenticate that a request was made by the holder of a wallet identity key — login being the most common example, but any action applies — in a single request, without a prior challenge round-trip and without a full mutual-authentication session.

The verifier first makes its identity public key available to clients. A client then produces an **authentication proof**: a small payload `{ action, identityKey, expiresAt, nonce }` together with a signature over that payload, created with the wallet's signing operation addressed to the verifier's identity key. When the authenticated request carries a payload of its own — for example a request body such as a new username on a profile update — that payload MAY be bound into the same signature, so the proof authenticates not merely the action but the specific contents of the request. The verifier checks the signature against the claimed identity key, checks that the proof is fresh (not expired), and records the proof's nonce so it can be used at most once. A valid proof authenticates the caller as the holder of `identityKey` for the stated `action`.

The proof is **signature-based** (asymmetric, so only the key holder can produce it), **expiry-bound** (replay is limited to a short window and verifier storage stays bounded), and **single-use** (replays within the window are rejected by a small, short-lived store).

## Motivation

Many applications need only a narrow guarantee: *prove, in a single request, that it was made by the holder of identity key K, recently, and only once* — for example, to authenticate a login. This is far less than an ongoing authenticated peer relationship, and benefits from a primitive scoped to exactly that need.

The peer-to-peer authentication standardised today, BRC-103, establishes a **mutual-authentication session**: a multi-message handshake (with optional certificate exchange) in which each side issues challenge nonces, the parties sign over them, and a session is tracked across subsequent requests. It is the right tool for an ongoing authenticated channel between peers. For a single, stateless action it is heavier than necessary, and its ready-made integrations are bound to a particular server environment (for example authenticated middleware for a specific web framework, or an authenticated socket transport) that maintains session state.

The challenge nonces used inside such a handshake — in the `@bsv/sdk` reference implementation, the `createNonce` / `verifyNonce` primitives that its BRC-103 `Peer` issues and checks — are **server-issued building blocks of the larger protocol**, made single-use by the protocol's session tracking and backed by separate digital signatures for authentication — they are not standalone login tokens. On their own they carry no expiry, are a symmetric keyed MAC rather than a signature over an application statement, and would require unbounded storage to be made single-use. A one-shot bearer proof is therefore a distinct need, which this standard addresses directly.

This standard defines that lightweight primitive: a single, self-contained **signed proof** that carries its own `action`, expiry, and unique nonce. It requires no handshake and no session, and it is independent of any server framework or transport — verification is a pure signature-and-freshness check plus one "has this nonce been used?" lookup, which an implementer backs with whatever store suits the deployment (an in-memory map, a database with a unique index, a key/value store such as Redis, and so on). It reuses the existing key-derivation and signing primitives (BRC-42, BRC-43, BRC-100), achieves the narrow guarantee — authenticated, recent, single-use — and is intended to complement BRC-103 for the many cases where a full mutual-authentication session is unnecessary.

## Terminology

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" are to be interpreted as described in RFC 2119.

- **Client** — the party proving control of an identity key (e.g. a browser wallet).
- **Verifier** — the party checking the proof (e.g. a backend server). The verifier also possesses a wallet with its own identity key.
- **Identity key** — a compressed (33-byte) secp256k1 public key identifying a party, hex-encoded.
- **Counterparty** — relative to a wallet performing a signing or verification operation, the *other* party's identity key, as defined by BRC-43. When the client signs, the counterparty is the verifier's identity key; when the verifier checks the signature, the counterparty is the client's identity key.
- **Authentication proof** — the object defined in this standard.

## Specification

### Proof structure

An authentication proof is a JSON object with two members:

| Field | Type | Description |
|---|---|---|
| `data` | object | The signed statement (see below). |
| `signature` | array of byte values (0–255) | A digital signature over the canonical encoding of `data`. |

`data` is a JSON object with the following members, all of which are signed:

| Field | Type | Description |
|---|---|---|
| `action` | string | The operation the proof authorizes, e.g. `"login"`. Letters, numbers, and spaces only. |
| `identityKey` | string | The client's identity public key (hex-encoded, compressed). The authenticated subject. |
| `expiresAt` | number | Absolute expiry as integer milliseconds since the Unix epoch. |
| `nonce` | string | A unique value: the base64 encoding of 32 cryptographically random bytes. |

All four `data` fields MUST be present and MUST be signed. `identityKey` is included in the signed statement so the signature attests to the complete claim ("I, `identityKey`, authorize `action` until `expiresAt` with `nonce`").

In addition to the four `data` fields, a proof MAY bind a **request payload** — application-defined bytes that accompany the request, such as the request body. The payload is **not** a member of `data` and is **not** carried inside the proof; it travels with the request as it normally would, and is bound only by being included in the signed bytes (see Canonical encoding). A proof that binds a payload and one that does not are distinct cases: a verifier that expects the request to carry a bound payload MUST verify against it, and a verifier that expects none MUST verify without one.

### Canonical encoding

The bytes that are signed and verified MUST be produced deterministically and identically by client and verifier as follows:

1. Form the string `S = action + "\n" + identityKey + "\n" + decimal(expiresAt) + "\n" + nonce`, where `"\n"` is the single line-feed character (U+000A) and `decimal(expiresAt)` is the base-10 integer string.
2. Encode `S` as UTF-8 to produce the byte sequence `H`.
3. If the proof binds **no** request payload, the bytes to sign and verify are exactly `H`.
4. If the proof binds a request payload `P` (a byte sequence), the bytes to sign and verify are `H`, followed by a variable-length integer (VarInt) encoding the byte length of `P`, followed by `P` itself.

The line-feed delimiter is unambiguous because none of the field values contain it (`identityKey` is hex, `expiresAt` is decimal digits, `nonce` is base64, and `action` is restricted to letters, numbers, and spaces). Implementations MUST reject or reformat any `action` that could contain the delimiter. The request payload is appended **length-prefixed rather than delimited**, so it may contain arbitrary bytes (including line feeds, NUL, or any binary content) without ambiguity, and the explicit length distinguishes an empty bound payload (length 0) from no bound payload (nothing appended).

How an application's payload is reduced to the byte sequence `P` is out of scope for this standard, but the reduction MUST be identical on client and verifier. The client MUST bind the exact bytes it transmits, and the verifier MUST bind the raw bytes it received — not a value re-parsed and re-serialized (e.g. a JSON object whose key order, whitespace, or encoding may differ from the bytes on the wire).

### Parameters

A deployment is parameterised by:

- **`protocol`** — a BRC-43 protocol identifier of the form `[2, name]`. Security level `2` binds derived keys to the counterparty. `name` consists of letters, numbers, and spaces. The same `protocol` MUST be configured on client and verifier.
- **`validityWindow`** — the maximum lifetime of a proof. RECOMMENDED default: 120000 ms (2 minutes).
- **`clockSkew`** — tolerance for clock differences. RECOMMENDED default: 30000 ms.

### Obtaining the verifier's identity key

Before a client can produce a proof it MUST know the verifier's identity public key. This key is the `counterparty` the client signs toward, and the same key's wallet performs verification. The verifier MUST make it available to clients out of band — for example served from a well-known endpoint or distributed in client configuration. The key is public; this step exchanges no secret. A client that signs toward the wrong key produces a proof the intended verifier cannot validate.

### Creating a proof (client)

Having obtained the verifier's identity key `verifierKey` (above), to authorize `action`:

1. Obtain the client's own identity key, `identityKey`.
2. Set `expiresAt = now + validityWindow` (current time in ms).
3. Generate `nonce` as base64 of 32 fresh random bytes.
4. Construct `data = { action, identityKey, expiresAt, nonce }`.
5. Produce `signature` by signing the canonical encoding of `data` — including the bound request payload, if this request carries one — with the wallet's BRC-100 signing operation, using:
   - `protocolID` = the configured `protocol`,
   - `keyID` = `nonce`,
   - `counterparty` = `verifierKey`.
6. Transmit `{ data, signature }` to the verifier, along with the request payload (if any) as the request normally carries it. The transmitted payload MUST be byte-for-byte the payload bound in step 5.

The signing key is derived per BRC-42/BRC-43 from the client's identity key, the verifier as counterparty, the `protocol`, and the `keyID`. Because the `keyID` is the per-request `nonce`, a **distinct child key is derived for every proof** — no signing key is reused across requests. The client transmits no private material.

### Verifying a proof (verifier)

Given a received proof and an `expectedAction`, the verifier MUST perform the following checks, and MUST reject the proof (treat it as unauthenticated) if any fails:

1. **Shape.** `data` is present with string `action`, non-empty string `identityKey`, finite numeric `expiresAt`, and non-empty string `nonce`; `signature` is an array.
2. **Action.** `data.action` equals `expectedAction`.
3. **Freshness.** `now < data.expiresAt` (not expired) **and** `data.expiresAt - now <= validityWindow + clockSkew` (not minted further into the future than the window allows — this prevents a client from issuing a long-lived proof).
4. **Signature.** The signature verifies over the canonical encoding of `data` — including the request payload bound from the raw bytes received with the request, when the action is expected to carry one — using the verifier's wallet BRC-100 verification operation with `protocolID` = the configured `protocol`, `keyID` = `data.nonce`, and `counterparty` = `data.identityKey`. A request whose received payload differs from the bound one (or that omits a payload the proof bound, or supplies one it did not) fails this check. Any error raised during verification (e.g. a malformed `identityKey` or signature) MUST be treated as a verification failure, not propagated as an error.
5. **Single-use.** `data.nonce` is recorded in a single-use store via an atomic insert-if-not-exists, keyed on the nonce; if the nonce is already present, the proof is rejected as a replay. The single-use check MUST occur only after the signature check passes, so that invalid proofs do not populate the store.

On success the caller is authenticated as `data.identityKey`. The verifier MUST treat `data.identityKey` as the authenticated subject. Applications that also receive an identity through another channel (e.g. a certificate subject, or a separate account field) SHOULD assert that it equals the authenticated `data.identityKey`.

### Single-use store

The store records consumed nonces and answers "was this nonce already used?".

- It MUST be **atomic**: a single conditional write (e.g. a uniqueness constraint), never a read-then-write, otherwise two concurrent requests bearing the same nonce can both be accepted.
- In multi-instance or serverless deployments it MUST be **shared** across instances (e.g. a database with a unique index). A per-process in-memory store is acceptable only for a single long-lived instance.
- Entries need only be retained until `expiresAt`; because expiry is enforced independently (step 3), an entry whose `expiresAt` has passed can be evicted safely (e.g. a TTL index). Storage therefore remains bounded to proofs seen within one `validityWindow`.

## Security Considerations

- **Asymmetric authentication.** The proof is a digital signature, so only the holder of `identityKey`'s private key can produce one that verifies under `counterparty = identityKey`. Tampering with `data.identityKey` changes the verification key and fails the signature check; the identity binding therefore does not depend on `identityKey` being among the signed bytes, though it is signed for completeness.
- **Replay.** Replay is bounded to `validityWindow` by the expiry check, and eliminated within that window by the single-use store. The two mechanisms are complementary: expiry bounds storage; single-use closes the residual window. A non-atomic or non-shared store reintroduces replay under concurrency or across instances (see Single-use store).
- **Restart / eviction.** Because the expiry lives in the signed statement, evicting expired entries — or losing an in-memory store on restart — re-opens replay for at most one `validityWindow`, never indefinitely.
- **Action binding.** A proof is valid only for the `expectedAction` the verifier checks against, so a proof minted for one operation (e.g. `login`) cannot be replayed against another (e.g. `delete`). Verifiers MUST check `action`.
- **Request-payload integrity.** When a payload is bound, the signature covers its exact bytes, so an intermediary that alters the request body — or replays a proof against a different body — fails verification. This binding is only as strong as the agreement on payload bytes (see Canonical encoding): the client MUST bind exactly the bytes it sends, and the verifier MUST bind the raw bytes it received. A proof that binds no payload makes no guarantee over the request body, so the payload SHOULD be bound for any action whose body is security-relevant (e.g. a field update).
- **Confidentiality.** The proof is integrity- and origin-protected, not confidential; `identityKey`, `action`, `expiresAt`, and `nonce` are transmitted in clear. Transport security (TLS) is assumed for confidentiality and to limit interception.
- **Clock dependence.** Freshness depends on reasonably synchronised clocks. `clockSkew` accommodates small differences; large client/verifier clock drift can cause false rejects or extend the effective window.
- **Protocol separation.** Distinct `protocol` names (or distinct verifier identity keys) domain-separate proofs between unrelated services, so a proof for one service cannot be presented to another.

## Implementations

A reference implementation in TypeScript, [`@bsv/auth`](https://www.npmjs.com/package/@bsv/auth), provides `createAuthProof` (client) and `verifyAuthProof` (verifier) over any BRC-100 wallet, with the single-use store injected by the consumer (e.g. a database TTL collection or an in-memory map). It exposes the parameters above with the recommended defaults. Both functions accept an optional request body, which it normalizes to bytes and binds into the signature as described above: a string is taken as UTF-8, an `ArrayBuffer` or typed array as raw bytes, and a plain object or any array as JSON. Omitting the body yields a bodyless proof whose signed bytes are exactly the canonical encoding of `data`.

## References

1. RFC 2119: Key words for use in RFCs to Indicate Requirement Levels — https://www.rfc-editor.org/rfc/rfc2119
2. BRC-42: BSV Key Derivation Scheme (BKDS) — https://github.com/bitcoin-sv/BRCs/blob/master/key-derivation/0042.md
3. BRC-43: Security Levels, Protocol IDs, Key IDs and Counterparties — https://github.com/bitcoin-sv/BRCs/blob/master/key-derivation/0043.md
4. BRC-100: Wallet-to-Application Interface — https://github.com/bitcoin-sv/BRCs/blob/master/wallet/0100.md
5. BRC-103: Peer-to-Peer Mutual Authentication and Certificate Exchange Protocol — https://github.com/bitcoin-sv/BRCs/blob/master/peer-to-peer/0103.md
6. BRC-31: Authrite Mutual Authentication — https://github.com/bitcoin-sv/BRCs/blob/master/peer-to-peer/0031.md
7. `@bsv/sdk`: TypeScript SDK — the `createNonce` / `verifyNonce` primitives and the BRC-103 `Peer` reference implementation — https://www.npmjs.com/package/@bsv/sdk
