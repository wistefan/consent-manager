# TICKET-011: Consent Signing and Key Management

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-011 |
| **Title** | Consent Signing and Key Management |
| **Priority** | High |
| **Complexity** | S/M (Small–Medium) |
| **Dependencies** | TICKET-001 |
| **Related User Stories** | US-CM-019, US-CM-020, US-PM-008 |

## Description

Sign consent receipts so that any party can verify a receipt is authentic and unaltered, and publish
the verification keys as a JWKS.

This ticket replaces the pre-revision "Consent Encryption and Key Management". The hybrid
AES-256-CBC + RSA scheme existed for one purpose: protecting consent payloads in transit to Data
Space Connectors. With data exchange out of scope there is no such transit, and encrypting data at
rest in a database the service itself must read would protect nothing.

What retains value is **signing**: a consent receipt travels outside this service, is retained by
users and participants, and may be presented later as evidence. It needs to be verifiable.

Complexity drops from M to S/M accordingly.

## Technical Details

### Signature Format

Receipts are signed as a **detached JWS** (RFC 7515) over the canonical JSON serialisation of the
receipt.

| Parameter | Value |
|-----------|-------|
| Algorithm | `ES256` (ECDSA P-256) preferred; `RS256` supported for compatibility |
| Header | `alg`, `kid`, `typ: "JOSE"` |
| Payload | Canonical JSON serialisation of the receipt (RFC 8785 JCS) |
| Serialisation | Detached — the signature travels beside the receipt, not wrapped around it |

Canonicalisation is not optional. Without a deterministic serialisation, a receipt that round-trips
through any JSON library will fail verification on key ordering or number formatting alone.

`ES256` is preferred over the original's RSA: smaller signatures, and no exposure to the padding
pitfalls of raw RSA operations.

### Signing Service

```java
public interface ConsentSigningService {
    SignedReceipt sign(ConsentReceipt receipt);
    boolean verify(ConsentReceipt receipt, String detachedJws);
}
```

`SignedReceipt` carries the receipt and its detached JWS. Signing failures **propagate** — they are
never swallowed and never yield an unsigned receipt presented as signed. The original swallowed
crypto errors and returned `undefined` (original bug 06-DE-12.3), so a caller could not distinguish
failure from success.

A signing failure on the grant path (TICKET-008) must fail the request. Recording a consent whose
receipt cannot be signed produces a record nobody can later prove.

### Key Management

Keys are supplied to the service as **mounted files or environment-provided PEM**, never committed:

| Configuration | Purpose |
|---------------|---------|
| `consent-manager.signing.active-key-id` | The `kid` used for new signatures |
| `consent-manager.signing.keys[].id` | Key identifier, published as `kid` |
| `consent-manager.signing.keys[].private-key-path` | PEM private key (active key only) |
| `consent-manager.signing.keys[].public-key-path` | PEM public key |
| `consent-manager.signing.keys[].algorithm` | `ES256` or `RS256` |

**Loading is eager and validated at startup**, in an injectable singleton — not lazily in a static
initialiser. A missing, unreadable or malformed key file fails startup with a message naming the
file and what was wrong with it. The original loaded key files synchronously at module import
(original bug 18.7), so a missing file crashed an unrelated import with an opaque error; making it a
deliberate, validated startup step is both clearer and fails in the right place.

Private key material is never logged, never returned by any endpoint, and never included in an error
message or stack trace. Add a test asserting that a startup failure message contains the file path
but no key content.

### Key Rotation

Rotation is supported by design, not deferred:

- The configuration holds a **list** of keys; the active one signs, all of them verify.
- Publishing several keys in the JWKS lets a rotated-out key keep verifying receipts already issued.
- A receipt's `kid` selects the verification key, so old receipts stay verifiable indefinitely.

Rotating is: add the new key, point `active-key-id` at it, redeploy, and retire the old key only
once no receipt needing it is still in circulation.

### JWKS Endpoint — `GET /.well-known/jwks.json` (US-PM-008)

**Auth:** none. The keys are public; that is the point.

Returns a standard JWKS containing the **public** halves of all configured keys, each with its
`kid`, `alg` and `use: "sig"`. Served outside the API context path, and cacheable
(`Cache-Control: public, max-age=3600`).

This replaces the pre-revision `GET /participants/consent-signature`, which returned a bare PEM
string: a JWKS carries key identifiers, supports rotation, and is consumable by any standard JOSE
library without bespoke parsing.

### Key Generation

Document the commands rather than shipping generation scripts — keys are an operational artefact,
not a build output:

```
# ES256 (preferred)
openssl ecparam -name prime256v1 -genkey -noout -out consent-signing.key
openssl ec -in consent-signing.key -pubout -out consent-signing.pub

# RS256 (compatibility)
openssl genrsa -out consent-signing.key 3072
openssl rsa -in consent-signing.key -pubout -out consent-signing.pub
```

Production key provisioning (Kubernetes Secrets, mounted volumes) is covered by TICKET-018.

### Files to Create

```
src/main/java/eu/prometheusx/consentmanager/
  signing/ConsentSigningService.java
  signing/JwsConsentSigningService.java
  signing/SigningKeyStore.java          # eager load, validate, select by kid
  signing/SigningConfiguration.java     # @ConfigurationProperties, validated
  signing/CanonicalJsonSerializer.java  # RFC 8785
  api/JwksController.java
```

## Acceptance Criteria

1. [ ] A receipt signs to a detached JWS carrying `alg` and `kid`
2. [ ] Sign → verify round-trips successfully
3. [ ] Verification fails when any byte of the receipt is altered
4. [ ] Verification fails for a signature made with a different key
5. [ ] Canonical serialisation is stable across key ordering and equivalent number formats
6. [ ] A receipt re-serialised through a different JSON library still verifies
7. [ ] Both `ES256` and `RS256` are supported and selected per configured key
8. [ ] Keys load eagerly at startup; a missing or malformed key fails startup with a descriptive message
9. [ ] Startup failure messages name the offending file but contain no key material
10. [ ] Signing errors propagate; no code path yields an unsigned receipt presented as signed
11. [ ] `GET /.well-known/jwks.json` returns valid JWKS containing only public keys
12. [ ] The JWKS includes every configured key so rotated-out keys still verify
13. [ ] A receipt signed with a retired key still verifies while that key remains published
14. [ ] The JWKS endpoint requires no authentication and is cacheable
15. [ ] No log line, error response or stack trace contains private key material
16. [ ] Parameterized tests cover the algorithm matrix and the tamper cases

## Removed Scope

| Removed | Reason |
|---------|--------|
| `encryptPayloadAndKey` (AES-256-CBC + RSA hybrid encryption) | Existed to protect consent in transit to DSCs; there is no such transit |
| `decryptPayloadAndKey` | Same |
| `AESKey.pem` and the `generateAES` script | No symmetric encryption remains |
| `GET /participants/consent-signature` returning a PEM | Replaced by a standard JWKS endpoint |
| US-DE-006 (end-to-end encrypted data exchange) | Data exchange is out of scope |

Original bugs 18.7 (keys loaded at module import) and 06-DE-12.3 (encryption errors swallowed) are
resolved by eager validated loading and by propagating signing failures.
