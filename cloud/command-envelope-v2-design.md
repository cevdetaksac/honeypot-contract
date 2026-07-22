# Command Envelope v2 + hardware-backed identity — design gate

> Contract: 1.4.5  
> Status: design-only; **not production wire**  
> Current production remains v1 HMAC + client 4.9.0 soft-allow/observe.  
> Client hello may advertise `caps.command_envelope_v2: "off"|"observe"` only
> (see `api/03-control-websocket.md`). Operator key distribution draft:
> [`operator-keyset-design.md`](./operator-keyset-design.md).

## Goals

- operator authorization remains verifiable if cloud routing is compromised;
- bind command to tenant, device, params, expiry, policy and approval set;
- reject replay while preserving idempotent retry/result behavior;
- allow future device payload encryption independently from authorization;
- support TPM-backed device proof and controlled re-enrollment.

## Candidate canonical envelope

```json
{
  "version": 2,
  "tenant_id": "tenant-uuid",
  "device_id": "device-uuid",
  "command_id": "command-uuid",
  "command_type": "network_restore",
  "params_hash": "sha256:<lowercase-hex>",
  "issued_at": "2026-07-22T00:00:00.000000Z",
  "expires_at": "2026-07-22T00:05:00.000000Z",
  "nonce": "<128-bit base64url>",
  "operator_id": "operator-uuid",
  "key_id": "operator-key-id",
  "policy_version": "policy-uuid",
  "approvals": [],
  "signature": "<base64url>"
}
```

## Decisions required before implementation

1. **Serialization:** canonical JSON (RFC 8785/JCS) vs deterministic CBOR.
2. **Algorithm:** Ed25519 vs WebAuthn assertion verification semantics.
3. **Signer custody:** browser/WebAuthn key vs managed signing service. If
   cloud-blind authorization is required, cloud cannot hold the operator
   private key.
4. **Approval policy:** command classes requiring one/two approvers; approval
   signatures cover the complete envelope hash and cannot be transplanted.
5. **Replay:** nonce + `command_id` durable replay window; duplicate delivery
   returns the same terminal result and never re-executes.
6. **Key lifecycle:** registration, rotation overlap, revocation, break-glass
   and tenant recovery.
7. **Confidentiality:** optional HPKE/hybrid encryption to a device key is a
   separate layer; routing metadata remains visible.

## TPM device identity candidate

- Device generates non-exportable signing key; cloud stores public key,
  attestation state and enrollment generation.
- Proof-of-possession binds `device_id`, tenant, nonce and current enrollment
  generation.
- MachineGuid remains compatibility identity during pilot; TPM absence is
  `unsupported`, not failure.
- Re-enrollment requires explicit owner approval, records old/new key ids and
  invalidates cloned enrollment generations.
- No irreversible hardware lock. Replacement motherboard/TPM clear has a
  documented recovery path.

## Rollout gates

1. Promote exact serialization, algorithm, errors and test vectors into
   `api/03-control-websocket.md`.
2. Implement cloud dual-read/dual-write; v1 remains available.
3. Client advertises capability and verifies v2 in observe mode.
4. Dashboard shows key/approval/replay/rollback status.
5. Internal pilot, downgrade and compromised-cloud tests pass.
6. Enforcement/floor change ships in a separate explicit contract release.

No code may emit a production `version:2` command before gate 1.
