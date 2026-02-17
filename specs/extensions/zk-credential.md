# x402 Extension: zk-credential

- **Extension ID:** `zk-credential`
- **x402 compatibility:** v2

## Summary

`zk-credential` enables **pay-once, redeem-many** access without introducing a stable session identifier (API keys/bearer tokens). After x402 settlement, the issuer issues a signed credential; later requests present a **zero-knowledge proof** that the client holds a valid, unexpired credential for the requested origin and tier.

**Non-goals:** Key discovery endpoints and on-the-wire keyset protocols are out of scope; this spec defines only presentation-time key carriage and verifier authorization requirements.

## Roles

- **Client**: pays once, stores credential, generates proofs for later requests.
- **Server**: advertises extension; forwards issuance input during settlement.
- **Issuer**: signs credentials. MAY be the Facilitator (default) or the Server itself; similar to how Server MAY equal Facilitator in x402.
- **Verifier**: the policy-enforcing component that validates proofs and authorizes access. Typically the Server, or an authorized gateway acting on its behalf. Verifiers accept only `service_id` + issuer key combinations they are independently configured to trust; extension advertisements are informational, not authoritative. The mechanism by which a verifier is authorized to enforce a service's policy (e.g., deployment topology, mTLS, static configuration) is out of scope.

## Transport and Encoding

### Issuance (Phase 1) — standard x402 extension plumbing
- Commitment travels inside `PaymentPayload.extensions["zk-credential"].info.commitment` via the standard `PAYMENT-SIGNATURE` header.
- Credential is returned inside `SettleResponse.extensions["zk-credential"].credential` via the standard `PAYMENT-RESPONSE` header.

### Presentation (Phase 2) — body envelope
- Proof and application payload are wrapped in an `x402_zk_credential` body envelope.
- Presence of `x402_zk_credential` in the request body is the canonical signal for ZK credential redemption.
- **Proofs MUST be in the request body** (UltraHonk proofs ~15KB exceed header limits).
- Redemption requests **MUST** use an HTTP method that permits a request body. `POST` is **RECOMMENDED**.
- **Zero custom headers** for the entire extension lifecycle.

### Content types
- `Content-Type: application/json` **REQUIRED**
- `Content-Type: application/cbor` **OPTIONAL**

### Canonical encoding for cryptographic objects

Suite-specific cryptographic objects (commitments, signatures) **MUST** use the suite-typed string format:

```
"<suite-id>:<base64url(bytes)>"
```

All public keys are encoded as raw bytes base64url (no padding) in a `*_pubkey` field and are interpreted under the suite indicated by the nearest `suite` / `*_suite` field. The encoding rules are:

- The pubkey field is base64url of raw public-key bytes, no padding.
- Key serialization (compressed/uncompressed, curve point encoding) is defined by the suite.
- Verifiers **MUST** reject keys that are not valid encodings for the suite.

For the `pedersen-schnorr-poseidon-ultrahonk` suite, points use uncompressed encoding: `0x04 || x || y` (64 bytes for BN254).

Examples:
- Adjacent-field public key: `"issuer_suite": "pedersen-schnorr-poseidon-ultrahonk", "issuer_pubkey": "BAAB..."`
- Suite-typed commitment: `"pedersen-schnorr-poseidon-ultrahonk:BAAB..."`
- Suite-typed signature: `"pedersen-schnorr-poseidon-ultrahonk:AQID..."`

Hex (`0x...`) is reserved for on-chain artifacts only (addresses, transaction hashes).

Other binary fields (`proof`, `origin_token`) use plain base64url without suite prefix.

## Phase 1 — Payment + credential issuance

1. Server responds `402` and advertises `extensions["zk-credential"]` with `{ info, schema }` structure.
2. Client retries payment with `PAYMENT-SIGNATURE`; includes commitment inside `PaymentPayload.extensions["zk-credential"].info.commitment`.
3. Server forwards `commitment` to the issuer during settlement.
4. Issuer returns settlement result plus a signed `credential`.
5. Server's `enrichSettlementResponse` hook injects credential into `SettleResponse.extensions["zk-credential"]`; client reads it from the standard `PAYMENT-RESPONSE` header.

## Phase 2 — Redemption

Client sends a request with an `x402_zk_credential` proof envelope in the request body. Server verifies locally, unwraps `payload` for the application handler, and serves the resource if valid.

Redemption requests **MUST** use an HTTP method that permits a request body. `POST` is **RECOMMENDED**.

## Wire format

### 1) Extension advertisement (in `PaymentRequired.extensions["zk-credential"]`)

Follows the x402 SDK `{ info, schema }` pattern:

```json
{
  "extensions": {
    "zk-credential": {
      "info": {
        "version": "0.1.0",
        "credential_suites": ["pedersen-schnorr-poseidon-ultrahonk"],
        "issuer_suite": "pedersen-schnorr-poseidon-ultrahonk",
        "issuer_pubkey": "BAAB...",
        "max_credential_ttl": 86400,
        "service_id": "k7VzM_xR9bQ2h1nPfEjw"
      },
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
          "commitment": {
            "type": "string",
            "description": "Suite-typed commitment: '<suite>:<base64url(point)>'"
          }
        }
      }
    }
  }
}
```

- `info.version` (REQUIRED)
- `info.credential_suites` (REQUIRED)
- `info.issuer_suite` (REQUIRED) — suite identifier string
- `info.issuer_pubkey` (REQUIRED) — base64url of raw public-key bytes (no suite prefix, no padding). This is a currently valid key; it is **informational** for clients. Verifiers authorize keys from their local key set, not from the advertisement (see §Verification Keys).
- `info.max_credential_ttl` (OPTIONAL)
- `info.service_id` (REQUIRED) — identifies the logical policy domain for which verifiers enforce rules, not a specific physical server or deployment. Encoded as base64url of 16 random bytes (128 bits), no padding. Issuers **MUST** generate `service_id` using a cryptographically secure RNG. `service_id` is stable for a service across key rotations unless the service intentionally changes identity. MUST match credential `service_id`.
- `schema` declares what the client appends inside `info` (the commitment)

### 2) Payment request with commitment (Phase 1)

Commitment is placed inside `PaymentPayload.extensions["zk-credential"].info.commitment`. Only the standard `PAYMENT-SIGNATURE` header is used:

```
PAYMENT-SIGNATURE: <base64 PaymentPayload>
```

The client echoes the server's extension and appends `commitment` inside `info`:

```json
{
  "info": {
    "version": "0.1.0",
    "credential_suites": ["pedersen-schnorr-poseidon-ultrahonk"],
    "issuer_suite": "pedersen-schnorr-poseidon-ultrahonk",
    "issuer_pubkey": "BAAB...",
    "commitment": "pedersen-schnorr-poseidon-ultrahonk:<base64url-commitment-point>"
  },
  "schema": { "..." }
}
```

- `commitment` (REQUIRED) — suite-typed string encoding the Pedersen commitment point

### 3) Credential issuance (inside `SettleResponse.extensions["zk-credential"]` via `PAYMENT-RESPONSE` header)

The credential is returned inside the standard `PAYMENT-RESPONSE` header. No custom response headers are used.

```
PAYMENT-RESPONSE: <base64 SettleResponse>
```

Decoded `SettleResponse.extensions["zk-credential"]`:

```json
{
  "credential": {
    "suite": "pedersen-schnorr-poseidon-ultrahonk",
    "service_id": "k7VzM_xR9bQ2h1nPfEjw",
    "tier": 1,
    "identity_limit": 1000,
    "expires_at": 1707004800,
    "commitment": "pedersen-schnorr-poseidon-ultrahonk:<base64url-commitment>",
    "signature": "pedersen-schnorr-poseidon-ultrahonk:<base64url-signature>"
  }
}
```

All fields above are **REQUIRED**.

- `identity_limit` — maximum number of distinct pseudonymous identities the credential holder may derive for rate limiting purposes. The circuit enforces that derivation index `i` satisfies `0 <= i < identity_limit`.

### 4) Redemption request envelope (Phase 2 request body)

Client wraps the proof in an `x402_zk_credential` body envelope:

```
POST /api/resource HTTP/1.1
Content-Type: application/json
```

```json
{
  "x402_zk_credential": {
    "version": "0.1.0",
    "suite": "pedersen-schnorr-poseidon-ultrahonk",
    "issuer_pubkey": "BAAB...",
    "proof": "<base64url-proof>",
    "current_time": 1707004800,
    "public_outputs": {
      "origin_token": "<base64url-origin-token>",
      "tier": 1
    }
  },
  "payload": null
}
```

- `x402_zk_credential`: `version`, `suite`, `issuer_pubkey`, `proof`, `current_time`, `public_outputs` are **REQUIRED**
- `issuer_pubkey`: base64url of raw public-key bytes (must match `suite`; verifier checks against authorized key set)
- `payload`: application request body (or `null` for requests with no body); server middleware unwraps this for the application handler

## Origin binding

Server derives an `origin_id` from the request URL to prevent cross-endpoint replay:

```
canonical_origin = canonicalize(request_url)
stringToField(s) = SHA-256(UTF-8(s)) mod p   (where p is the BN254 scalar field order)
origin_id = stringToField(canonical_origin)
```

### Canonicalization algorithm

Given a request URL, produce `canonical_origin` via the following deterministic steps:

1. **Parse** the URL into RFC 3986 components: `scheme`, `host`, `port`, `path`. **Reject** if parsing fails.
2. **Scheme**: lowercase (e.g. `HTTPS` → `https`).
3. **Host**: lowercase. Implementations **MUST** convert Unicode hostnames to Punycode (IDNA) before lowercasing.
4. **Port**: omit default ports (`80` for `http`, `443` for `https`); include non-default ports.
5. **Path**: if empty, set to `/`.
6. **Dot-segment removal**: normalize `.` and `..` segments per RFC 3986 §5.2.4.
7. **Query and fragment**: **MUST** be excluded.
8. **Percent-encoding**: use the path as produced by the URL parser after dot-segment removal; do **not** decode/re-encode percent escapes.
9. **Assemble**: `canonical_origin = scheme + "://" + host + port_suffix + normalized_path` where `port_suffix` is `":" + port` only when non-default.

### Test vectors

| Input URL | `canonical_origin` |
|---|---|
| `https://API.Example.COM/v1/data` | `https://api.example.com/v1/data` |
| `https://api.example.com:443/v1/data` | `https://api.example.com/v1/data` |
| `http://api.example.com:8080/v1/data` | `http://api.example.com:8080/v1/data` |
| `https://api.example.com` | `https://api.example.com/` |
| `https://api.example.com/a/b/../c` | `https://api.example.com/a/c` |
| `https://api.example.com/a/./b` | `https://api.example.com/a/b` |
| `https://api.example.com/v1/data?key=val#frag` | `https://api.example.com/v1/data` |
| `https://api.example.com/hello%20world` | `https://api.example.com/hello%20world` |

## Proof statement

A valid proof MUST prove (suite-defined construction) that:
- the client holds an issuer-signed credential for `service_id`
- the credential was signed by the `issuer_pubkey` provided in the presentation
- `current_time <= expires_at`
- credential `tier` ≥ the server's required tier for the requested resource
- the client's chosen derivation index `i` satisfies `0 <= i < identity_limit`
- `origin_token` is deterministically derived from private credential material, `origin_id`, and derivation index `i`
- `origin_id` is correctly bound (prevents replay across origins)
- the proof outputs include `(origin_token, tier)`

Verifiers **MUST** verify the proof using the provided `issuer_pubkey` and **MUST** ensure that the key is authorized for the associated service (e.g., by matching against a locally configured allowlist or trusted key set).

Suites define the proving system and verifier parameters; SNARK and STARK suites are both compatible with this extension.

## Version and suite negotiation

- For `0.x` versions, client and server MUST match `version` exactly.
- Client MUST present a `suite` listed in the server's `credential_suites`.

## Clock skew check

Server MUST reject before verification if:

```
abs(current_time - server_clock) > 60 seconds
```

## Replay prevention / rate limiting

`origin_token` is a pseudonymous, origin-bound identifier derived within the proof from private credential material, the origin binding, and a derivation index. Using the same derivation index across requests produces a stable `origin_token` (enabling rate limiting) at the cost of cross-request linkability within that origin; different indices produce unlinkable tokens, up to the credential's `identity_limit`.

Verifiers MAY use `origin_token` for bounded replay detection and/or rate limiting within the credential validity window. Caches MUST be TTL-bounded by credential expiry.

## Errors

| Code | HTTP | Meaning |
|------|------|---------|
| `credential_missing` | 402 | no payment or credential provided |
| `tier_insufficient` | 402 | credential tier does not meet the server's required tier; may indicate a server–issuer tier configuration mismatch |
| `unsupported_version` | 400 | version not supported |
| `unsupported_suite` | 400 | suite not supported |
| `invalid_proof` | 400 | proof verification failed (includes origin mismatch) |
| `payload_too_large` | 413 | proof body exceeds server limit |
| `unsupported_media_type` | 415 | content-type not supported |
| `rate_limited` | 429 | origin token rate limited |

## Verification Keys: Distribution & Rollover

- Presentations **MUST** include `issuer_pubkey`.
- Verifiers **MUST** only accept presentations whose `issuer_pubkey` is authorized by verifier policy (e.g., local configuration, trusted key list) for the `service_id`.
- Issuers **MAY** rotate keys at any time; verifiers **SHOULD** overlap old and new keys long enough to avoid breaking valid credentials before expiry.

## Tier Coordination

The `tier` field is an ordinal integer that flows from issuance through proof verification. The server and issuer **MUST** share a common understanding of `tier` values for a given `service_id`, including which payment amounts correspond to which tiers. Verification treats `tier` as ordinal: a credential is accepted when its `tier` ≥ the server's required tier for the requested resource.

The mechanism by which the server and issuer establish this mapping (e.g., static configuration, a registration API, shared deployment) is out of scope for this specification. Implementations **SHOULD** document their tier coordination approach.

