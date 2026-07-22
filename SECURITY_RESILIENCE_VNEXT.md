# Security & Resilience vNext — Shared Delivery Plan

> Contract planning version: **1.4.6**
> Baseline: client **4.9.0**, production floor remains **4.9.0**  
> Audience: Windows client, Cloud/API, Dashboard and QA implementers  
> Status: **planning + promoted observe schemas** (enforce still off)

This document coordinates parallel client and cloud work. Observe field shapes
for the P1 package are now normative in `api/08-architecture.md`,
`api/01-auth.md`, `api/03-control-websocket.md` and related agent docs.
Enforcement, enrollment, and offline ingest endpoints remain gated.

Client-side analysis and complete backlog:
`cloud-client/docs/SECURITY_RESILIENCE_ROADMAP.md`.

## 1. Non-negotiable invariants

These rules apply to every workstream:

1. Network Guard remains **alert-only**. Detection never automatically
   suspends/kills a process or restores networking. `suspend_process`,
   `kill_process` and `network_restore` remain exact-target/confirm-gated.
2. Motor/Guardian stand down only for legitimate update handoff, signed
   operator PIN stop, or uninstall. Other exits are tamper + recovery.
3. No `REALTIME_PRIORITY_CLASS`, covert DNS/ICMP tunnel, automatic
   admin/domain lockout, or irreversible hardware fingerprint lock.
4. New sensors begin in shadow mode. No fleet-wide enforcement before
   false-positive, resource and event-loss measurements pass.
5. Endpoint/banner/detection-flow secrecy is not a security boundary.
6. No password, token, private key, typed text, TURN credential or decoy
   credential in logs/audits/errors.
7. Cloud must dual-read/dual-write during migrations; old clients continue to
   operate until the declared fleet floor is raised.

Canonical references:

- [`agent/network-guard.md`](agent/network-guard.md)
- [`agent/persistence-and-tamper.md`](agent/persistence-and-tamper.md)
- [`api/03-control-websocket.md`](api/03-control-websocket.md)
- [`api/04-self-update.md`](api/04-self-update.md)
- [`api/08-architecture.md`](api/08-architecture.md)

## 2. Shared execution protocol

Each work item follows this order:

1. **Contract promotion:** final field names, endpoint reuse, auth, error codes,
   min client/cloud version, rollback and acceptance are added to the relevant
   normative MD.
2. **Cloud compatibility first:** cloud accepts/adds the new shape without
   requiring it from old clients.
3. **Client capability:** client advertises support and runs shadow/observe
   mode; destructive enforcement remains off.
4. **Dashboard visibility:** health, rollout state, errors and operator action
   are visible before enforcement.
5. **Measured pilot:** internal hosts → opt-in tenant → staged fleet.
6. **Enforcement:** only after compatibility metrics are green; raise floor
   separately.

Every result must include:

- unit + integration + downgrade/rollback tests;
- CPU/RAM/disk/network/event-loss budget;
- audit/redaction tests;
- offline/reboot/update/Session-0 scenarios;
- idempotency and replay behavior;
- feature flag and emergency disable.

## 3. Current gaps (baseline 4.9.0)

| Area | Existing | Gap |
|---|---|---|
| Command integrity | HMAC helper + whitelist/expiry/confirm | Missing signature is still accepted during transition |
| Artifact trust | Optional SHA-256 | No mandatory Authenticode signing or WinVerifyTrust |
| Resilience | Motor + Guardian + watchdog/task | Restart SLO/storm breaker and `installed-but-not-running` repair need strengthening |
| Identity events | Security Event Log sensor | Password change/reset events 4723/4724 absent |
| Ransomware | Canary + behavior + approved suspend | No ETW file-I/O sensor/event-loss telemetry |
| Reverse engineering | PyInstaller/NSIS | No embedded-secret CI scan, signed runtime manifest or measured Nuitka/native-core decision |
| Deception | Protocol banners/handshakes | No bounded tarpit/fingerprint-resistance program |
| Device identity | MachineGuid | No TPM-backed key/attestation/re-enrollment |
| E2E | TLS + shared HMAC | No operator asymmetric signature or per-device payload encryption |
| OOB | Signed network baseline/restore | No legitimate alternate path; no durable offline urgent queue |

## 4. Parallel delivery lanes

### Lane A — Client can start immediately

- Guardian resilience metrics, restart-storm breaker and repair tests.
- ETW read-only/shadow PoC and event-loss measurement.
- Event Log 4723/4724 parsing/unit tests.
- Embedded-secret scan, Nuitka benchmark and hybrid native-core PoC.
- Runtime Authenticode/module-manifest design.
- Canary/fingerprint-resistance and bounded-tarpit PoCs.

No new wire field from this lane ships until promoted below.

### Lane B — Cloud can start immediately

- Sign every newly queued/pushed command using the existing v1 HMAC shape.
- Record command-signing coverage without requiring enforcement yet.
- Prepare artifact signer/checksum metadata and rollout status storage.
- Prepare additive health/event ingestion for resilience, Event Log and ETW
  shadow telemetry.
- Add dashboard rollout/status surfaces; no destructive automation.
- Prepare canonical command-envelope-v2 and TPM/WebAuthn design documents.

### Lane C — Joint design before coding

- Mandatory command-signing rollout and key derivation migration.
- Authenticode publisher trust/rotation/recovery.
- Asymmetric operator command signature + multi-party approval.
- Per-device encryption/TPM identity and re-enrollment.
- Alternate management path/OOB customer model.

## 5. P0 shared work packages

### ZT-600 — Make command signatures mandatory

**Risk:** client currently accepts a command when `signature` is absent.

Current v1 compatibility algorithm:

```text
secret = SHA256(token + "|" + COMPUTERNAME + "|yesnext-chp-v1")
message = command_id + "|" + command_type + "|" + issued_at
signature = HMAC-SHA256(secret, message)
```

Before enforcement, Cloud and Client must publish matching deterministic test
vectors for casing, machine-name source, timestamp serialization and command
field aliases.

**Cloud/API**

- Sign commands created by REST, dashboard actions, scheduled jobs and control
  WS pushes. A single queue row must retain one stable `issued_at`.
- Return the exact same signed fields through pending poll and WS.
- Track `signed_commands_total`, `unsigned_commands_total`,
  `signature_generation_failed`.
- Do not dispatch a destructive command when signature generation fails.
- Keep signing enabled for old clients; old clients ignore unknown fields.

**Client**

- Add observe mode: verify signature, report missing/invalid, but keep existing
  behavior during compatibility measurement.
- Add enforce mode: missing or invalid signature → reject, do not execute,
  submit a redacted audit/result.
- The local config cannot disable enforcement after fleet policy requires it.
- Preserve command expiry, whitelist and confirmation checks; signature does
  not replace them.

**Dashboard**

- Per-client signing state: `legacy | observing | enforcing | degraded`.
- Block high-impact actions if cloud cannot sign.
- Display rejected command id/type/reason without params secrets.

**Rollout**

1. Cloud signs 100% of commands.
2. Client observe release; measure at least one fleet cycle.
3. Resolve machine-name/test-vector mismatches.
4. Enable enforce by capability/fleet policy.
5. Raise production floor only in a later contract.

**Acceptance**

- REST and WS commands produce identical validation.
- Missing/modified/replayed signature fails in enforce mode.
- Old client still receives commands during cloud-first phase.
- No token/signing secret appears in logs or API responses.

### SUP-001 / SUP-001b — Authenticode + WinVerifyTrust

**Cloud/API**

- Release metadata stores SHA-256, expected publisher identity/certificate
  thumbprint set, signing timestamp policy and artifact size.
- Update API never publishes an unsigned artifact as production.
- Support signer rotation with an overlap window (current + next publisher).
- Dashboard shows `signed`, `publisher_valid`, `checksum_valid`, rollout/error.

**Client/build**

- Timestamp-sign installer and executable; CI verifies after signing.
- Updater checks WinVerifyTrust/publisher allowset **before execution**, then
  SHA-256. Either failure aborts safely and reports a redacted reason.
- Guardian/runtime integrity verifies installed binary before recovery launch;
  tamper mismatch must not create a restart storm.
- Define offline verification, expired signing cert and timestamp behavior.

**Acceptance**

- Bit-flipped, unsigned, wrong-publisher and expired-without-valid-timestamp
  artifacts are rejected.
- Valid signer rotation and downgrade/rollback are tested.
- Existing update handoff produces no false tamper.

### SR-001/002/003 — Resilience SLO and repair

**Client health additions (draft; promote before implementation)**

```json
{
  "resilience": {
    "guardian_installed": true,
    "guardian_running": true,
    "guardian_exit_code": 0,
    "daemon_restarts_24h": 0,
    "guardian_restarts_24h": 0,
    "last_recovery_ms": 1840,
    "restart_backoff_sec": 0,
    "restart_storm": false,
    "binary_integrity": "valid|invalid|unknown"
  }
}
```

**Client**

- Self-heal `service_installed=true && service_ok=false`.
- Bounded exponential backoff under repeated crash; watchdog never spins.
- Persist boot/session-safe counters and distinguish update/PIN/uninstall.
- Fault-injection tests: kill, task delete/disable, service stop, corrupt
  config, disk full, reboot during update.

**Cloud/Dashboard**

- Additive health ingestion; unknown/missing means legacy, not failure.
- Alert on sustained degraded/restart-storm, with dedupe.
- Show last recovery duration, restart count and legitimate stand-down reason.
- Never issue automatic process/network containment from resilience telemetry.

### ID-401 — Password change/reset visibility

**Client**

- Add Security Event IDs `4723` (password change attempt) and `4724` (password
  reset attempt) with actor, target, host/domain, result and timestamp.
- Do not collect old/new password or credential material.
- Normalize events through the existing security event path.

**Cloud/API**

- Accept additive normalized event types; dedupe by client/event-record/time.
- Correlate burst by tenant/actor/target/domain in a configurable window.
- Retention/redaction follows existing `SecurityEvent` policy.

**Dashboard**

- Identity incident view: actor, targets, count, source host/DC and timeline.
- Alert-only. No automatic admin/domain lockout.
- Future response opens a separately confirmed external IAM playbook.

### RANS-301 — ETW file-I/O shadow sensor

**Client**

- ETW PoC observes create/write/rename/delete; correlate with
  `pid + image + path + process_start_time`.
- Report buffer pressure/dropped events/provider restart and fallback state.
- Correlate with canary, VSS, extension/entropy shift and suspicious origin.
- Shadow mode only; no automatic suspend/kill.

**Cloud/API**

- Additive batch ingestion for aggregated windows, not raw unbounded file
  events. Define payload/retention limit before promotion.
- Store sensor health separately from detection outcome.
- Build FP corpus labels for backup/indexer/compiler/browser/AV workloads.

**Dashboard**

- Shadow sensor badge, event-loss warning and correlation evidence.
- High-confidence result may offer existing confirm-gated exact-target Suspend;
  it never auto-executes.

**Acceptance**

- Event loss is visible; silent loss cannot look healthy.
- CPU/RAM/IO budget passes representative server workload.
- Existing Network Guard hard alert-only invariant remains unchanged.

### REV-101/102/104 — Binary exposure and runtime integrity

**Client/build**

- CI scans source, PyInstaller archive, NSIS and final binary for long-lived
  secrets/private keys/test credentials.
- Produce SBOM, dependency versions, artifact digest and build provenance.
- Compare PyInstaller vs Nuitka using startup/RAM/CPU/size/AV/update/crash data.
- Evaluate a small native security core for verification/integrity/ETW/device
  key work; do not rewrite the whole product without measured benefit.
- No anti-debugger, self-modifying packer or analysis-tool sabotage.

**Cloud**

- Release record stores provenance/SBOM digest and artifact signing state.
- Never treat endpoint/banner secrecy as security.
- Move policy that can safely remain server-side into signed/versioned bundles;
  agent must validate before applying.

**Dashboard**

- Release trust view: signer, checksum, SBOM/provenance and deployment state.

### ZT-601 — Canonical asymmetric command envelope (design package)

This P0 item is design-only; HMAC enforcement ships first.

Required envelope concepts:

```json
{
  "version": 2,
  "tenant_id": "...",
  "device_id": "...",
  "command_id": "...",
  "command_type": "...",
  "params_hash": "sha256:...",
  "issued_at": "...",
  "expires_at": "...",
  "nonce": "...",
  "operator_id": "...",
  "key_id": "...",
  "policy_version": "...",
  "signature": "..."
}
```

**Joint decisions**

- Browser/WebAuthn vs managed signing service; cloud must not hold an operator
  private key if the goal is cloud-blind authorization.
- Canonical JSON/CBOR serialization and signature algorithm.
- Admin public-key registration, rotation, revocation and break-glass.
- Multi-party approval for panic/isolate/credential/restore classes.
- Replay windows, idempotent result and audit hash.
- Confidentiality is separate: optional payload encryption to a device public
  key (HPKE/hybrid); routing metadata remains visible to cloud.

No implementation begins until these choices are promoted to
`api/03-control-websocket.md`.

## 6. P1 work packages

After P0 compatibility and telemetry are green:

- **RES-103/105/106:** signed heartbeat, SACL access audit and ACL drift.
  Cloud stores offender evidence and dedupes tamper incidents.
- **DEC-201/202/205/206/208/209:** canary coverage, controlled NTFS bait,
  bounded protocol tarpit and fingerprint-resistance. Cloud distributes
  signed/versioned profiles; Dashboard exposes emergency disable/resource use.
- **RANS-302/303:** ETW event-loss + correlation model and FP evaluation.
- **NET-501/502:** restore dry-run/diff/rollback. Cloud command remains
  confirm-gated; Dashboard shows exact planned changes before approval.
- **OOB-501:** DPAPI/integrity-protected offline urgent queue with idempotent
  replay. Cloud dedupes event ids and reports last acknowledged sequence.
- **ID-402/403:** password-change burst correlation and identity timeline.
- **ZT-602/603/605b:** hardware-backed operator signing, agent key-set
  verification and TLS/interception/cloud-compromise test matrix.
- **DEV-601:** TPM device-key PoC, proof-of-possession and controlled
  re-enrollment design.

## 7. P2 controlled pilots

- Opt-in credential honeytoken (no LSASS/real credential injection).
- Customer-managed secondary HTTPS/VPN/NIC/cellular path; no covert tunnel.
- Nuitka and hybrid native-core release experiment.
- E2E encrypted sensitive command params.
- TPM device identity pilot and clone/re-enrollment workflow.

P2 is tenant opt-in and cannot raise the global floor by itself.

## 7A. P1 client observe package — 2026-07-22 (schemas promoted 1.4.5)

The client has landed the following **default-off / observe-only primitives**.
Exact wire shapes are now in canonical API/agent docs. Production floor
remains **4.9.0**; no enforce toggles are on:

- RES-103 candidate signed heartbeat (`heartbeat_proof`, HMAC v1) —
  `api/01-auth.md`;
- RES-105/106 path-free DACL fingerprints + drift (`access_integrity`) —
  `api/08-architecture.md`;
- RANS-302/303 bounded ETW loss/pressure + correlation (`etw_shadow`) —
  `api/08-architecture.md`;
- DEC-201/202 `canary_coverage` + DEC-205/206/208/209 `deception_health[]`;
- NET-501/502 `network_restore` `dry_run` / `rollback_version` —
  `api/03-control-websocket.md` + `agent/network-guard.md`;
- OOB-501 DPAPI+HMAC local queue primitive (**ingest endpoint not promoted**);
- ID-402/403 `event_log_health.password_burst` (wire name SoT; not
  `identity_burst`);
- ZT-602/603 public operator-key metadata scaffold (**GET endpoint + verify
  not promoted**);
- DEV-601 read-only TPM probe (`device_identity`).

**Still cloud/dashboard gated before enabling flags / pilots:**

1. Persist/display the promoted observe blocks; missing = `legacy`. **(done)**
2. Offline urgent-event ingest + ACK/idempotency (OOB-501) — **promoted
   1.4.7** → `api/10-offline-urgent-queue.md`; client flag still default off.
3. Operator public-key endpoint + algorithm + test vectors before ZT-603
   verify. (observe stub GET exists; verify stays false)
4. TPM enrollment/PoP/re-enrollment UX (DEV-601 beyond probe).
5. Dashboard observe cards + emergency-disable before tenant pilot. **(cards done)**
6. Keep enforce / auto-contain / auto-lockout / Guardian reject-stale **off**.

## 8. Cloud implementation checklist

Cloud agents should execute in this order:

1. Read this file, then the canonical references in §1.
2. Implement command signing for **all** creation/delivery paths in observe
   compatibility mode; add deterministic tests.
3. Add additive persistence/health models for signing and resilience.
4. Add release signer/checksum/provenance fields; do not require them from
   current 4.9.0 clients yet.
5. Accept 4723/4724 normalized events and ETW shadow aggregates only after the
   final schemas are promoted.
6. Build dashboard status surfaces before any enforcement toggle.
7. Keep all destructive actions confirm-gated and Network Guard alert-only.
8. Publish cloud implementation notes back to `CHANGELOG.md`.

## 9. Client implementation checklist

Client agents should execute in this order:

1. Keep roadmap work feature-flagged and backward-compatible.
2. Add signing observe telemetry; do not enforce until cloud coverage is 100%.
3. Build Guardian resilience/fault-injection tests.
4. Add 4723/4724 and ETW shadow sensor independently.
5. Add Authenticode/WinVerifyTrust and secret/provenance checks to build/update.
6. Run Nuitka/native-core experiments as separate artifacts; do not replace the
   production build from benchmark assumptions.
7. Publish exact fields/limits/errors into canonical contract before merge.

## 10. Joint release gates

- Contract and deterministic crypto test vectors merged first.
- Cloud compatibility deployed before client enforcement.
- Dashboard exposes status/error/rollback before rollout.
- Client/cloud unit, integration and downgrade tests green.
- Internal pilot has no restart storm, missed update or destructive FP.
- Security logs pass secret/redaction tests.
- Rollback does not require unsigned commands or an unsigned artifact.
- Production floor changes in a separate explicit contract release.

