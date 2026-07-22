# Operator public keyset — design (ZT-602/603)

> Status: **design / non-normative**  
> Contract: 1.4.5  
> Client scaffold: `cloud-client/client_operator_keys.inspect_keyset` —
> public metadata validation only; **`verify_enabled` always false**.

## Goal

Distribute tenant operator public keys so agents can verify asymmetric
command envelopes (v2) without cloud holding the private key.

## Proposed endpoint (future promote)

`GET /api/agent/operator-keys` (Bearer device token):

```json
{
  "keys": [
    {
      "key_id": "current",
      "algorithm": "ed25519",
      "public_key": "<base64url-or-spki>",
      "state": "active"
    },
    {
      "key_id": "next",
      "algorithm": "ed25519",
      "public_key": "<…>",
      "state": "next"
    },
    {
      "key_id": "old",
      "algorithm": "ed25519",
      "public_key": "<…>",
      "state": "revoked"
    }
  ],
  "rotation_policy": {
    "overlap_required": true,
    "max_active": 2
  }
}
```

`state`: `active` | `next` | `revoked`. Private material in any field →
client rejects the document (`private_material_rejected`).

## Client inspect result (local today)

```json
{
  "mode": "observe",
  "verify_enabled": false,
  "valid": true,
  "active_keys": 2,
  "revoked_keys": 1,
  "rotation_overlap": true,
  "private_material_rejected": false,
  "errors": []
}
```

## Gates before verify_enabled

1. Canonical serialization + algorithm decision in
   `command-envelope-v2-design.md`.
2. Deterministic verify test vectors.
3. Dual-key rotation + revocation UX on dashboard.
4. Hello `caps.command_envelope_v2` may be `"observe"`; `"enforce"` never
   selected by client until fleet pilot completes.
5. HMAC v1 soft-allow remains until signed coverage is measured.

## Non-goals now

- WebAuthn ceremony details (dashboard-owned)
- Agent-side key generation for operators
- Multi-party approval (ZT-604) — separate
