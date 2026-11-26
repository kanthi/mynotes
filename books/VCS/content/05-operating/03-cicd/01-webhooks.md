---
title: Webhooks and Event-Driven CI
---

# Webhooks and Event-Driven CI

Polling a Git forge every 30 seconds across hundreds of repositories wastes API quota and adds latency to every build. **Webhooks** invert that model: when a `push`, `pull_request`, or `release` event occurs, the forge sends an HTTP `POST` request with a JSON payload to your receiver endpoint within milliseconds.

## Anatomy of a Reliable Webhook Receiver

A production webhook handler cannot just parse JSON and trigger a build—it must satisfy four operational invariants:

1. **Cryptographic signature verification:** Every delivery includes an HMAC-SHA256 header computed over the raw request body using a shared signing value configured on both the forge and the receiver. Always compare digests with constant-time comparison (`hmac.compare_digest`) to prevent timing side channels.
2. **Fast asynchronous acknowledgment:** Return `202 Accepted` within 1–2 seconds after enqueueing the event. If your receiver runs a 5-minute build inline, the forge times out the HTTP connection and retries the delivery repeatedly.
3. **Idempotent deduplication:** Forges guarantee *at-least-once* delivery. Store the unique delivery ID header (for example, `X-GitHub-Delivery` or `X-Gitlab-Event-UUID`) in Redis or PostgreSQL with a TTL so retried webhooks do not launch duplicate deployments.
4. **Payload replay protection:** Reject deliveries outside an acceptable clock skew window or re-fetch authoritative state from the forge API when making high-trust deployment decisions.

## Minimal Python HMAC-SHA256 Verifier

```python
import hashlib
import hmac


def verify_webhook_signature(raw_body: bytes, header_sig: str, signing_key: bytes) -> bool:
    """Verify a forge HMAC-SHA256 webhook signature in constant time."""
    if not header_sig.startswith("sha256="):
        return False
    expected = "sha256=" + hmac.new(signing_key, raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, header_sig)


if __name__ == "__main__":
    sample_body = b'{"ref":"refs/heads/main","after":"a1b2c3d"}'
    key = b"local-dev-signing-value"
    valid_header = "sha256=" + hmac.new(key, sample_body, hashlib.sha256).hexdigest()

    assert verify_webhook_signature(sample_body, valid_header, key) is True
    assert verify_webhook_signature(sample_body + b" ", valid_header, key) is False
    print("Signature verification checks passed.")
```

> [!WARNING]
> Always compute the HMAC over the **exact raw request bytes**, never over a re-serialized JSON object. Key ordering or whitespace differences after `json.loads` → `json.dumps` will invalidate every legitimate signature.

## Hands-On Lab: Test Signature Verification and Tamper Detection

Run the Python verifier locally and confirm it accepts an intact payload while rejecting a single-byte modification:

```bash
python3 - <<'PY'
import hashlib, hmac

key = b"lab-demo-key"
payload = b'{"event":"push","repository":"desk-api","ref":"refs/heads/main"}'
sig = "sha256=" + hmac.new(key, payload, hashlib.sha256).hexdigest()

def verify(body, header):
    expected = "sha256=" + hmac.new(key, body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, header)

print("Original payload valid:", verify(payload, sig))
print("Tampered payload valid:", verify(payload.replace(b"main", b"prod"), sig))
PY
```
