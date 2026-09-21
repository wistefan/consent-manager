# TICKET-011: Consent Encryption and Key Management

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-011 |
| **Title** | Consent Encryption and Key Management |
| **Priority** | High |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-001 |
| **Related User Stories** | US-CM-020, US-DE-006 |

## Description

Implement the hybrid encryption scheme (AES-256-CBC + RSA) used to encrypt consent data for secure transmission between the Consent Manager and Data Space Connectors. This also covers key generation scripts and key management.

## Technical Details

### Encryption Scheme Overview

The Consent Manager uses a hybrid encryption approach:
1. **AES-256-CBC** encrypts the consent payload (symmetric, fast for large data)
2. **RSA** encrypts the AES key (asymmetric, allows secure key exchange)

This ensures that only the holder of the RSA private key can decrypt the AES key, and only with the AES key can the consent payload be decrypted.

### Encryption Function: `encryptPayloadAndKey`

**File:** `src/libs/crypto/encryption.ts` (or within consent controller)

**Input:**
```typescript
{
  payload: object;      // Consent data to encrypt
  publicKey: string;    // RSA public key (PEM format) of the recipient
}
```

**Algorithm:**
1. Generate a random AES-256 key (32 bytes)
2. Generate a random initialization vector (IV, 16 bytes)
3. Serialize the payload to JSON string
4. Encrypt the JSON payload with AES-256-CBC using the key and IV
5. Encrypt the AES key with RSA using the recipient's public key
6. Return the encrypted payload, encrypted key, and IV

**Output:**
```typescript
{
  encryptedPayload: string;  // Base64-encoded AES-encrypted consent data
  encryptedKey: string;      // Base64-encoded RSA-encrypted AES key
  iv: string;                // Base64-encoded initialization vector
}
```

### Decryption Function: `decryptPayloadAndKey`

**Input:**
```typescript
{
  encryptedPayload: string;  // Base64-encoded
  encryptedKey: string;      // Base64-encoded
  iv: string;                // Base64-encoded
  privateKey: string;        // RSA private key (PEM format)
}
```

**Algorithm:**
1. Decrypt the AES key with RSA using the private key
2. Decrypt the payload with AES-256-CBC using the decrypted key and IV
3. Parse the JSON string back to an object
4. Return the decrypted payload

### Consent Signing

**File:** `src/libs/crypto/signing.ts`

The Consent Manager signs consent data with its RSA private key to prove authenticity:

1. Serialize consent to JSON
2. Sign with RSA-SHA256 using the consent signature private key
3. Return the base64-encoded signature

Participants can verify the signature using the public key from `GET /participants/consent-signature`.

### Key Management

**Directory:** `src/config/keys/`

**Key Files:**
| File | Format | Purpose |
|------|--------|---------|
| `consentSignature.pem` | RSA Private Key (PEM) | Sign consent data |
| `consentSignature.pub.pem` | RSA Public Key (PEM) | Verify consent signatures (distributed to participants) |
| `AESKey.pem` | AES Key (Base64) | Default AES key for local operations |

**Key Loading:**

Load keys lazily on first use, not synchronously at module import time (fixing original bug 18.7):

```typescript
let _privateKey: string | null = null;

function getPrivateKey(): string {
  if (!_privateKey) {
    _privateKey = readFileSync(resolve(__dirname, '../config/keys/consentSignature.pem'), 'utf-8');
  }
  return _privateKey;
}
```

Add error handling: if a key file does not exist, throw a descriptive error rather than crashing the entire module.

### Key Generation Scripts

**NPM Scripts:**

| Script | Command | Description |
|--------|---------|-------------|
| `generatePrivateKey` | `openssl genrsa -out src/config/keys/consentSignature.pem 2048` | Generate RSA private key |
| `generatePublicKey` | `openssl rsa -in src/config/keys/consentSignature.pem -pubout -out src/config/keys/consentSignature.pub.pem` | Extract public key |
| `generateAES` | Node.js script: `crypto.randomBytes(32)` | Generate AES-256 key |

### Error Handling

The original `encryptPayloadAndKey` swallows errors (bug 12.3). The reimplementation must:
1. Catch encryption errors
2. Log the error with details
3. **Re-throw** the error so callers can handle it
4. Never return `undefined` on failure

### Files to Create

```
src/libs/crypto/
  encryption.ts          # encryptPayloadAndKey, decryptPayloadAndKey
  signing.ts             # signConsent, verifySignature
  keyManager.ts          # Lazy key loading with error handling
  index.ts               # Re-export
src/config/keys/
  .gitkeep               # Placeholder (actual keys not committed)
scripts/
  generateAESKey.ts      # AES key generation script
```

## Acceptance Criteria

1. [ ] `encryptPayloadAndKey` encrypts a payload with AES-256-CBC and the AES key with RSA
2. [ ] `decryptPayloadAndKey` successfully decrypts the output of `encryptPayloadAndKey`
3. [ ] Round-trip test: encrypt → decrypt produces the original payload
4. [ ] RSA signing produces a verifiable signature
5. [ ] Key loading is lazy (not at module import time)
6. [ ] Missing key files produce descriptive errors, not crashes
7. [ ] Encryption errors are logged and re-thrown (not swallowed)
8. [ ] Key generation scripts work correctly (`generatePrivateKey`, `generatePublicKey`, `generateAES`)
9. [ ] Unit tests cover encryption, decryption, signing, and error cases
10. [ ] Keys directory has `.gitkeep` but actual keys are in `.gitignore`

## Known Issues to Address

1. **Key files loaded at module import** (original bug 18.7): Use lazy loading with descriptive error messages.
2. **Encryption error swallowed** (original bug 12.3): Re-throw errors after logging. Never return `undefined`.
3. **No key rotation support**: Consider adding key versioning or rotation mechanism as a future enhancement (document but do not implement).
