---
title: Post-Quantum Crypto, HPKE (RFC 9180), and FIPS 140-3 (Go 1.24–1.27)
---

# Post-Quantum Crypto, HPKE (RFC 9180), and FIPS 140-3 (Go 1.24–1.27)

Between Go 1.24 and Go 1.27, the standard library overhauled its cryptographic stack for post-quantum defense, zero-dependency FIPS 140-3 compliance, and automatic secret scrubbing:

| Package | Standard | Capabilities | Introduced |
|---|---|---|---|
| `crypto/mlkem` | FIPS 203 | ML-KEM-768, ML-KEM-1024 (18% faster in Go 1.26) | Go 1.24 |
| `crypto/mldsa` | FIPS 204 | ML-DSA-44, ML-DSA-65, ML-DSA-87 signatures | Go 1.27 |
| `crypto/hpke` | RFC 9180 | Hybrid Public Key Encryption | Go 1.26 |
| `crypto/hkdf` / `crypto/sha3` | RFC 5869 / FIPS 202 | Key derivation & SHA-3 / SHAKE (`hash.Cloner` in 1.25) | Go 1.24 |
| `crypto/fips140` | FIPS 140-3 | Built-in certified crypto core (`WithoutEnforcement` in 1.26) | Go 1.24 |
| `runtime/secret` | Security | Stack/register zeroing (child goroutines inherit in 1.27) | Go 1.26 (`GOEXPERIMENT=runtimesecret`) |

## Scenario 1: End-to-End Post-Quantum Key Exchange (`mlkem` + `hkdf` + `NewGCMWithRandomNonce`)

```go
package main

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/hkdf"
	"crypto/mlkem"
	"crypto/sha256"
	"fmt"
)

func main() {
	dk, err := mlkem.GenerateKey768()
	if err != nil {
		panic(err)
	}
	peerEK, _ := mlkem.NewEncapsulationKey768(dk.EncapsulationKey().Bytes())
	sharedSender, ct := peerEK.Encapsulate()
	_, _ = dk.Decapsulate(ct)

	aesKey, _ := hkdf.Key(sha256.New, sharedSender, nil, "settlement-v1", 32)
	block, _ := aes.NewCipher(aesKey)
	aead, _ := cipher.NewGCMWithRandomNonce(block)

	sealed := aead.Seal(nil, nil, []byte("WIRE_TRANSFER:5000000"), []byte("aad"))
	opened, _ := aead.Open(nil, nil, sealed, []byte("aad"))
	fmt.Println("Decrypted PQ Payload:", string(opened))
}
```

## Scenario 2: Zero-Trust Queue Encryption with `crypto/hpke` (Go 1.26+)

```go
package main

import (
	"crypto/hpke"
	"fmt"
)

func main() {
	kem := hpke.DHKEM_X25519_HKDF_SHA256
	kdf := hpke.HKDF_SHA256
	aead := hpke.AES_256_GCM
	privKey, pubKey, _ := kem.GenerateKeyPair()

	encapKey, sender, _ := hpke.NewSender(pubKey, kdf, aead, []byte("pii-ingest-v1"))
	ct, _ := sender.Seal([]byte("topic:kyc"), []byte(`{"id":"1234"}`))

	recipient, _ := hpke.NewRecipient(encapKey, privKey, kdf, aead, []byte("pii-ingest-v1"))
	pt, _ := recipient.Open([]byte("topic:kyc"), ct)
	fmt.Println("HPKE Decrypted:", string(pt))
}
```

## Scenario 3: FIPS 140-3 Selective Bypass and Post-Quantum TLS 1.3

```go
package main

import (
	"crypto/fips140"
	"crypto/md5"
	"crypto/tls"
	"fmt"
)

func ComputeLegacyS3ETag(payload []byte) [16]byte {
	var sum [16]byte
	// Go 1.26+: Safely bypass GODEBUG=fips140=only strictly for non-security ETags.
	fips140.WithoutEnforcement(func() { sum = md5.Sum(payload) })
	return sum
}

func NewPostQuantumTLSConfig() *tls.Config {
	return &tls.Config{
		MinVersion: tls.VersionTLS13,
		CurvePreferences: []tls.CurveID{
			tls.X25519MLKEM768,
			tls.SecP256r1MLKEM768,
			tls.MLKEM1024, // Go 1.27
		},
	}
}

func main() {
	fmt.Printf("FIPS Enabled=%t Enforced=%t Version=%s\n",
		fips140.Enabled(), fips140.Enforced(), fips140.Version())
	fmt.Printf("ETag=%x Curves=%d\n", ComputeLegacyS3ETag([]byte("data")), len(NewPostQuantumTLSConfig().CurvePreferences))
}
```

## Related Chapters

- [999d crypto/mldsa](999d-crypto-mldsa.md)
- [999e crypto/mlkem](999e-crypto-mlkem.md)
- [179 runtime/secret Zeroization](../22-security-hardening/179-runtime-secret-zeroization.md)
