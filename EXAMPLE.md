# Example: Using Cloudflare RFC 3161 TSA

## Quick Test

```bash
# Hash some data
DATA_HASH=$(echo -n "important document" | sha256sum | cut -d' ' -f1)

# Get cryptographic timestamp
curl -X POST https://rfc3161-tsa.infinitumnihil.workers.dev/tsr \
  -d "$DATA_HASH"
```

**Response:**
```json
{
  "status": "granted",
  "token": {
    "tsa_name": "Cloudflare RFC 3161 TSA",
    "hash": "916f0027a575074ce72a331777c3478d6513f786a591bd892da1a577bf2335f9",
    "timestamp": "2026-01-13T05:18:19.261Z"
  },
  "signature": "ek3kjLAVnv9op9xiKT0PjoLDid5z8UZ02UjC7ylxJ2FIzX5zPPp0...",
  "algorithm": "Ed25519"
}
```

**What this proves:** Your data existed at 2026-01-13 05:18:19 UTC (cannot be backdated)

## Integration with Multi-Authority Verification

```typescript
// Use 3 timestamp authorities for stronger proof
const AUTHORITIES = [
  'https://freetsa.org/tsr',                      // FreeTSA (free)
  'https://rfc3161-tsa.infinitumnihil.workers.dev/tsr',  // Cloudflare (free, <50ms)
  'http://timestamp.globalsign.com/tsa/r6advanced1'      // GlobalSign (free)
];

// Request from all in parallel
const timestamps = await Promise.all(
  AUTHORITIES.map(url =>
    fetch(url, { method: 'POST', body: dataHash })
      .then(r => r.json())
  )
);

// Check consensus (all within 1 second)
const times = timestamps.map(t => new Date(t.token.timestamp).getTime());
const maxDeviation = Math.max(...times) - Math.min(...times);

console.log('Consensus:', maxDeviation < 1000 ? 'VERIFIED' : 'FAILED');
console.log('Deviation:', maxDeviation, 'ms');
```

## Verification Example

```typescript
// Verify a timestamp signature
async function verifyTimestamp(timestampResponse) {
  // Import TSA's public key
  const publicKeyBytes = Uint8Array.from(
    atob(timestampResponse.tsa_certificate),
    c => c.charCodeAt(0)
  );

  const publicKey = await crypto.subtle.importKey(
    'spki',
    publicKeyBytes,
    { name: 'Ed25519' },
    false,
    ['verify']
  );

  // Verify signature
  const tokenBytes = new TextEncoder().encode(
    JSON.stringify(timestampResponse.token)
  );

  const signatureBytes = Uint8Array.from(
    atob(timestampResponse.signature),
    c => c.charCodeAt(0)
  );

  const valid = await crypto.subtle.verify(
    'Ed25519',
    publicKey,
    signatureBytes,
    tokenBytes
  );

  return valid;
}

// Test verification
const isValid = await verifyTimestamp(timestampResponse);
console.log('Timestamp signature valid:', isValid);
```

## Use Cases

**1. Code Signing**
```bash
# Timestamp your git commit
git rev-parse HEAD | xargs -I {} curl -X POST https://rfc3161-tsa.infinitumnihil.workers.dev/tsr -d {}

# Proves: This commit existed at timestamp (cannot claim earlier build date)
```

**2. Document Notarization**
```bash
# Timestamp contract
sha256sum contract.pdf | cut -d' ' -f1 | xargs -I {} curl -X POST https://rfc3161-tsa.infinitumnihil.workers.dev/tsr -d {} > contract-timestamp.json

# Proves: Contract existed at this time (backdating impossible)
```

**3. AI Decision Audit Trail** (Patent 99TB-414505)
```typescript
// Hash AI conversation
const conversationHash = await sha512(JSON.stringify({
  user: "Should I approve this loan?",
  ai_thinking: "Credit score 720, income verified...",
  ai_response: "Recommend approval"
}));

// Get multi-authority timestamps
const timestamps = await requestMultiAuthority(conversationHash);

// Store with blockchain anchor
await anchorToStellar(conversationHash, timestamps);

// Result: Tamper-proof audit trail for regulatory compliance
```

## Latency Comparison

| Timestamp Authority | Typical Latency |
|---------------------|----------------|
| FreeTSA.org | 1-3 seconds |
| GlobalSign | 2-5 seconds |
| DigiCert | 1-4 seconds |
| **Cloudflare RFC 3161** | **<50ms** |

**Why faster:**
- No cross-internet latency (Cloudflare's global network)
- Durable Object keeps key in memory
- Web Crypto API (native performance)
- Service binding available (RPC vs HTTP)

## Security Model

**Trust Considerations:**

**Own deployment:** You control the TSA (good for development, not for legal proof)

**Public instance (rfc3161-tsa.infinitumnihil.workers.dev):**
- Operated by third party
- Useful for testing
- NOT recommended for legal/compliance (you don't control key rotation)

**Best practice for legal proof:**
- Use 2-3 independent TSAs (FreeTSA + GlobalSign + this)
- Require consensus (all within 1 second)
- At least 1 must be established authority (FreeTSA/GlobalSign)

**This TSA is useful for:**
- Development/testing (free, fast)
- Cost optimization (1 free TSA in your 3-authority setup)
- Latency optimization (<50ms vs seconds)
- Fallback when external TSAs are down

---

**Deploy your own:** Takes 30 seconds with the button in README
