# x402 Extension: zk-credential

- **Extension ID:** `zk-credential`
- **x402 compatibility:** v2

## Summary

`zk-credential` enables **pay-once, redeem-many** access without introducing a stable session identifier (API keys/bearer tokens). After x402 settlement, the facilitator issues a signed credential; later requests present a **zero-knowledge proof** that the client holds a valid, unexpired credential for the requested origin and tier.

## Roles

- **Client**: pays once, stores credential, generates proofs for later requests.
- **Server**: advertises extension; forwards issuance input during settlement; verifies proofs locally.
- **Facilitator (Issuer)**: settles payment and issues the credential.

## Transport and Encoding

### Transport
- **Proofs MUST be supported in the HTTP request body.** Body transport is the REQUIRED conformance mode.
- **Servers MUST NOT require proofs in headers.**
- **Servers MAY accept a header-carried proof as an optimization, but MUST SUPPORT body transport for conformance.** 
- **Any header-carried proof is an optional, non-normative optimization.**
- **If any header is used, it MUST be metadata-only (e.g., `suite`, `kid`) and MUST NOT be required.**
- Credentials **MUST** be returned in the HTTP **response body**.

### Content types
- `Content-Type: application/json` **REQUIRED**
- `Content-Type: application/cbor` **OPTIONAL**

### Encoding rules
Encoding is determined by `Content-Type`:
- If `application/json`, all binary fields **MUST** be **base64url without padding** (RFC 4648 URL-safe alphabet, `-` and `_`, no trailing `=`).
- If `application/cbor`, binary fields are raw byte strings.

Timestamps:
- `expires_at` and `current_time` **MUST** be Unix time in seconds (integer).

Binary fields (JSON / base64url-no-pad):
- `proof`, `signature`, `pubkey`, `commitment`, `origin_token`, `service_id` (and any other byte arrays).

## Phase 1 — Payment + credential issuance

1. Server responds `402` and advertises `extensions.zk-credential`.
2. Client retries payment and includes `extensions.zk-credential.commitment`.
3. Server forwards `commitment` to the facilitator during settlement.
4. Facilitator returns settlement result plus a signed `credential`.
5. Server returns `credential` to the client in the response body.

## Phase 2 — Redemption

Client sends a `zk-credential` proof envelope in the request body. Server verifies locally and serves the resource if valid.

Redemption requests **SHOULD** use `POST` (proof in body). `GET`-with-body is not required and may be unsupported by intermediaries.

## Wire format

### 1) Extension advertisement (in 402 response body)

```json
{
  "extensions": {
    "zk-credential": {
      "version": "0.1.0",
      "credential_suites": ["<suite-id>"],
      "facilitator_pubkey": "<suite-id>:<pubkey-bytes>",
      "max_credential_ttl": 86400,
      "content_types": ["application/json", "application/cbor"]
    }
  }
}
```

- `version` (REQUIRED)
- `credential_suites` (REQUIRED)
- `facilitator_pubkey` (REQUIRED)
- `max_credential_ttl` (OPTIONAL)
- `content_types` (OPTIONAL): advertised supported content types

### 2) Payment request with commitment (Phase 1)

```json
{
  "x402Version": 2,
  "payment": { "...": "..." },
  "extensions": {
    "zk-credential": {
      "commitment": "<suite-id>:<base64url-commitment>"
    }
  }
}
```

- `commitment` (REQUIRED)

### 3) Credential issuance (Phase 1 response body)

```json
{
  "zk-credential": {
    "credential": {
      "suite": "<suite-id>",
      "kid": "key-YYYY-MM",
      "service_id": "<base64url-service-id>",
      "tier": 1,
      "identity_limit": 1000,
      "expires_at": 1707004800,
      "commitment": "<base64url-commitment>",
      "signature": "<base64url-signature>"
    }
  }
}
```

All fields above are **REQUIRED** except `kid`, which is **OPTIONAL** (supports key rotation; see Key rotation section).

### 4) Redemption request envelope (Phase 2 request body)

```json
{
  "zk-credential": {
    "version": "0.1.0",
    "suite": "<suite-id>",
    "kid": "key-YYYY-MM",
    "proof": "<base64url-proof>",
    "current_time": 1707004800,
    "public_outputs": {
      "origin_token": "<base64url-origin-token>",
      "tier": 1
    }
  }
}
```

- `version`, `suite`, `proof`, `current_time`, `public_outputs` are **REQUIRED**
- `kid` is **RECOMMENDED**

## Origin binding 

Server derives an `origin_id` from the request URL to prevent cross-endpoint replay:

```
canonical_origin = scheme + "://" + lowercase(host) + normalized_path
stringToField(s) = SHA-256(s) mod p   (where p is the BN254 scalar field order)
origin_id = Poseidon(stringToField(canonical_origin))
```

Normalization:
- scheme lowercase
- host lowercase; include port only if non-default
- strip trailing `/` from path
- exclude query string

## Proof statement

A valid proof MUST prove (suite-defined construction) that:
- the client holds a facilitator-signed credential for `service_id`
- `current_time <= expires_at`
- credential `tier` satisfies server policy
- `origin_id` is correctly bound (prevents replay across origins)
- the proof outputs include `(origin_token, tier)`

Suites define the proving system and verifier parameters; SNARK and STARK suites are both compatible with this extension.

## Clock skew check

Server MUST reject before verification if:

```
abs(current_time - server_clock) > 60 seconds
```

## Replay prevention / rate limiting

Servers use `origin_token` to enforce replay prevention and/or rate limiting; caches MUST be TTL-bounded by credential expiry.

## Errors

| Code | HTTP | Meaning |
|------|------|---------|
| `credential_missing` | 402 | no payment or credential provided |
| `tier_insufficient` | 402 | proof tier below requirement |
| `unsupported_suite` | 400 | suite not supported |
| `invalid_proof` | 400 | proof verification failed |
| `payload_too_large` | 413 | proof body exceeds server limit |
| `unsupported_media_type` | 415 | content-type not supported |
| `rate_limited` | 429 | origin token rate limited |

## Key rotation

Servers MUST support issuer key selection by `kid` (or a configured default if omitted).

Servers SHOULD expose issuer public keys via HTTP at:

`GET /.well-known/zk-credential-keys`

```json
{
  "keys": [
    { "kid": "key-YYYY-MM", "suite": "<suite-id>", "pubkey": "<base64url-pubkey>", "valid_from": 1706918400, "valid_until": null }
  ]
}
```

Clients MUST NOT depend on discovery; keys may also be provisioned out-of-band.

## Conformance

An implementation conforms if it:
1) advertises `extensions.zk-credential`  
2) accepts proofs in request bodies  
3) returns credentials in response bodies  
4) forwards `commitment` to facilitator during settlement  
5) verifies proofs locally during redemption  
6) computes `origin_id` per the normalization rules above  
7) enforces clock skew check  
8) implements the error codes above  
