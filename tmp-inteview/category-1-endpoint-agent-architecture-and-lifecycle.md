# Category 1 — Endpoint Agent Architecture and Lifecycle

## How to answer almost every question

Use this structure:

1. **Define the problem.**
2. **Give the design.**
3. **Explain the failure/security trade-off.**

### The one-sentence mental model

> An endpoint agent is a **headless, OS-supervised process** that performs **local enforcement**, communicates securely with a central server, stores durable state locally, and continues operating safely when the network or a non-critical component fails.

### Six ideas to repeat throughout the interview

**Supervision — Security — Separation — State — Shutdown — Recovery**

- **Supervision:** use SCM, systemd, or launchd.
- **Security:** use least privilege, protected credentials, authenticated IPC, and mTLS.
- **Separation:** separate communication, policy, enforcement, and privileged operations.
- **State:** use versioned, durable, atomic local state.
- **Shutdown:** stop gracefully and never leave a partial policy update.
- **Recovery:** restart safely, use backoff, rollback, and offline mode.

---

## Fundamentals and architecture

### 1. What is an endpoint agent? How is it different from a desktop application or backend service?

An **endpoint agent** is a program installed on a device that runs continuously as a **Windows Service**, **Linux daemon**, or **macOS launchd job**. It communicates with a central management server but performs important work locally: enforcement, monitoring, inventory, telemetry, or updates.

Compared with a desktop application, it is **headless** and must work without a logged-in user or GUI. Compared with a backend service, it runs on an endpoint that may be offline, heterogeneous, user-accessible, and potentially hostile.

**Say these keywords:** **long-running**, **headless**, **OS-supervised**, **local enforcement**, **offline operation**, **least privilege**.

### 2. Design a cross-platform endpoint agent that communicates with a central management server

I would create a shared core and thin OS-specific adapters.

#### Shared core

- **Lifecycle/state machine**: installed, unenrolled, enrolling, active, degraded, updating, stopping.
- **Communication client**: TLS/mTLS, retry, backoff, heartbeat, commands, and telemetry.
- **Policy manager**: authenticate, validate, version, and atomically activate policy.
- **Enforcement engine**: evaluate the active policy and apply local actions.
- **Durable store**: identity, policy snapshot, queue, cursors, and update state.
- **Update and diagnostics**: signed updates, rollback, logs, metrics, and health.

#### OS-specific adapters

- Windows: **SCM**, service accounts, DPAPI, named pipes/RPC.
- Linux: **systemd**, UID/GID, capabilities, Unix sockets.
- macOS: **launchd**, Keychain, XPC, entitlements, and TCC.

The enforcement loop must not wait for the server. It should use the **last-known-good policy** when disconnected.

**Good short answer:**

> “I would use a shared platform-independent core for transport, policy, state, and lifecycle, with OS-specific adapters for service management, secure storage, and enforcement. Communication would use mTLS, durable queues, bounded retries, and cached policy so a server outage does not stop local protection.”

### 3. How would you separate the enforcement, communication, and policy layers?

Use three clear responsibilities:

| Layer | Responsibility |
|---|---|
| **Communication** | Connect, authenticate, send/receive messages, retry, queue, and acknowledge |
| **Policy management** | Validate signatures/schema/version, compile, persist, activate, and roll back policy |
| **Local enforcement** | Observe the device, evaluate the active policy, and apply local actions |

The flow should be:

```text
server message
  -> authenticate and validate
  -> compile candidate policy
  -> persist it atomically
  -> activate it
  -> enforcement reads the new immutable snapshot
```

Never modify the live policy halfway through validation. Use **immutable snapshots**, **copy-on-write**, or **prepare/commit/rollback**.

**Keywords:** **separation of concerns**, **last-known-good policy**, **atomic activation**, **trust boundary**.

### 4. What are the major components of an endpoint agent?

| Component | Responsibility |
|---|---|
| **Service entry point** | Connect to the OS manager and report ready/stopped |
| **Communication client** | TLS/mTLS, commands, telemetry, retry, and backoff |
| **Enrollment/identity** | Generate keys, obtain credentials, rotate, and revoke |
| **Policy manager** | Validate, version, persist, and activate policy |
| **Policy evaluator** | Make deterministic decisions from policy and local facts |
| **Enforcement adapters** | Apply decisions to OS resources |
| **Local state store** | Persist snapshots, queues, cursors, and update state |
| **Privileged broker** | Perform only narrowly defined elevated operations |
| **Updater** | Verify, stage, activate, and roll back releases |
| **Diagnostics** | Logs, metrics, health, crash data, and support information |

The important part is not naming classes. It is showing **clear boundaries**, **least privilege**, and **failure isolation**.

### 5. How would you design an agent that runs continuously without a GUI?

I would make it a **headless foreground process** supervised by the native OS manager.

1. Register it as a Windows Service, systemd service, or launchd job.
2. Run it under a **dedicated least-privilege account**.
3. Use absolute paths and protected machine-wide data directories.
4. Report explicit startup/readiness status.
5. Handle stop, shutdown, reload, reboot, sleep, and network loss.
6. Send logs to the platform logging system with bounded local diagnostics.
7. Expose status through authenticated local IPC, not an open unauthenticated port.

If a UI is needed, use a separate **user-session process** that communicates with the service through secured IPC. Do not put UI inside a privileged service.

### 6. Single process versus multiple processes

#### Single process

**Pros:** simpler deployment, lower overhead, simpler shared state and debugging.

**Cons:** one crash can stop everything; privileged and unprivileged code share one boundary; a stuck component can block the whole agent; larger compromise **blast radius**.

#### Multiple processes

**Pros:** **fault isolation**, separate privileges, separate resource limits, and independent restart.

**Cons:** more IPC, packaging, monitoring, upgrade, and state-coordination complexity.

For a security-sensitive agent, I would usually separate:

- an unprivileged communication/policy process;
- a small privileged enforcement or broker process;
- optionally an updater and a user-session UI.

I would split processes only where there is a real **security boundary**, **fault boundary**, or independent lifecycle requirement.

### 7. How would you separate privileged and unprivileged operations?

Use a small **privileged broker** with a narrow typed IPC interface.

The unprivileged process handles networking, parsing, policy compilation, telemetry, and most business logic. The privileged process performs only operations that truly require elevation.

The IPC boundary must have:

- OS-level peer authentication;
- restrictive endpoint ACLs/ownership;
- an allowlist of operations;
- strict input validation in the privileged process;
- per-request authorization, deadlines, and rate limits;
- audit logging without secrets;
- no arbitrary command execution or arbitrary path access.

Do not trust a request simply because it came from “our” process. IPC is an **attack surface**.

**Strong phrase:** “Keep the privileged code small, typed, authenticated, authorized, and independently restartable.”

### 8. How should the agent manage configuration, local state, and credentials?

Keep them separate:

| Data | Example | Handling |
|---|---|---|
| **Configuration** | Server URL, feature flags, log level | Schema-versioned protected config |
| **Durable state** | Active policy, queue, cursors, update journal | Transactional database or atomic files |
| **Credentials** | Private key, certificate, bootstrap token | OS secure store or tightly protected key file |
| **Ephemeral state** | Connections, caches, worker queues | Recreate safely after restart |

Important rules:

- Use **least-privilege filesystem permissions**.
- Prefer DPAPI/Credential Manager, Keychain, or an appropriate Linux secret store.
- Never log tokens, private keys, or sensitive policy contents.
- Use schema versions and explicit migrations.
- Write state to a temporary location, flush if durability matters, then atomically rename/commit.
- Keep the old **last-known-good policy** until the new one is fully validated and persisted.
- Rotate credentials and support expiry, revocation, and re-enrollment.
- Bound queue and log size so disk usage cannot grow forever.

---

## Lifecycle and reliability

### 9. Complete lifecycle: installation to uninstallation

Remember: **Install → Enroll → Run → Update → Remove**.

#### Install

- Verify package **signature**, version, and architecture.
- Create the service/daemon registration and protected directories.
- Create the service identity and permissions.
- Install initial configuration and state schema.

#### Enroll

- Generate/load a device key pair.
- Authenticate the bootstrap request.
- Obtain a certificate or long-term credential.
- Persist identity atomically.
- Download and validate the initial policy.

#### Run

- Start automatically through the OS manager.
- Enforce the active policy locally.
- Maintain the server connection or polling schedule.
- Persist telemetry, commands, policy version, and health.

#### Update

- Download and verify a signed package.
- Stage it beside the current version.
- Activate atomically.
- Verify health and roll back if startup fails.

#### Remove

- Authorize uninstall.
- Stop gracefully.
- Revoke identity if required.
- Remove service registration, binaries, IPC endpoints, and temporary state.
- Decide explicitly whether logs/evidence are retained or deleted.

**Keywords:** **state machine**, **signed package**, **atomic persistence**, **rollback**, **authenticated uninstall**, **no orphaned privileged process**.

### 10. How does first-time enrollment establish agent identity?

Use a locally generated key pair and **proof of possession**.

1. The installer provides a short-lived bootstrap token or MDM-provisioned enrollment secret.
2. The agent generates a key pair locally; the private key never leaves the device.
3. The agent sends a CSR or signed request containing the public key and device metadata.
4. The server validates the token, scope, expiry, and replay protection.
5. The server binds the public key to a device record and issues a certificate.
6. The agent stores it securely and uses **mTLS** for future communication.

Support **rotation**, **revocation**, expiry, re-enrollment, and partial-failure recovery.

Do not use a MAC address, hostname, or hardware fingerprint as the sole identity. Those values can change or be spoofed. Use a cryptographic key and a server-issued device ID.

### 11. How does the agent start automatically after reboot?

Use the native service manager:

| OS | Mechanism |
|---|---|
| Windows | **SCM** service with automatic/delayed start, dependencies, and recovery |
| Linux | **systemd** unit enabled for the appropriate target |
| macOS | **launchd** LaunchDaemon/LaunchAgent with `RunAtLoad`/`KeepAlive` as appropriate |

Also verify that startup:

- does not require a logged-in user or GUI;
- uses the intended service identity;
- uses absolute paths;
- waits for required dependencies;
- reports readiness only after initialization;
- has bounded crash recovery.

Do not depend on login scripts, cron, or a desktop application for a machine-wide agent.

### 12. What happens when the agent crashes?

The OS manager detects the exit and applies a **bounded restart policy**. On restart, the agent reconstructs itself from durable state.

A good design includes:

- restart on unexpected failure;
- exponential backoff and restart-rate limits;
- crash dumps, exit codes, and diagnostics;
- recovery of the last-known-good policy and durable queues;
- idempotent startup and migrations;
- safe/degraded mode after repeated failures;
- alerting when the server becomes reachable again.

Also handle a hung process with a **health check/watchdog**; a process that is alive is not necessarily healthy.

Do not restart forever without limits. A deterministic bad policy or bad update may need **quarantine and rollback**, not another immediate restart.

### 13. How does the agent work when the management server is unavailable?

Treat the server as a source of updates and coordination, not as a synchronous dependency for every local decision.

- Keep a **last-known-good policy snapshot** locally.
- Continue local enforcement using that snapshot.
- Define explicit **fail-open/fail-closed** behavior for each control.
- Use a policy lease/expiry if stale policy must eventually stop being trusted.
- Queue telemetry in a bounded durable outbox.
- Retry with exponential backoff and jitter.
- Resume using policy versions, cursors, and idempotent request IDs.
- Report a local **degraded** health state.
- Reconcile state when the server returns.

The network must never block the local enforcement loop. The correct design is **offline-first degradation**, not “keep trying the request forever.”

### 14. How do you prevent multiple agent instances?

Use two layers:

1. The OS service manager owns the expected service instance.
2. The process acquires a protected singleton:
   - Windows named mutex;
   - Linux `flock`/protected Unix socket;
   - macOS launchd ownership plus a lock if needed.

If the lock is unavailable, log that another instance owns the role and exit cleanly.

Do not rely only on a PID file because PIDs can be reused and files can remain after a crash. The lock location must be protected from untrusted users.

For a multi-process agent, use separate role locks and let the supervisor own the process topology.

### 15. Graceful shutdown during a policy update

Use a **quiescing state** and transactional updates.

1. Receive the stop/shutdown request.
2. Enter `QUIESCING`.
3. Stop accepting new commands and new policy work.
4. Let the current update finish within the deadline, or cancel at a safe checkpoint.
5. Persist either the old policy or the fully committed new policy—never a partial one.
6. Persist update stages such as `DOWNLOADED`, `VERIFIED`, `STAGED`, and `COMMITTED`.
7. Flush essential telemetry within a bounded deadline.
8. Stop workers and transport, release IPC, and report stopped.

Use **atomic activation**: validate and persist the candidate policy separately, then switch one active-version pointer. On restart, inspect the journal and complete or roll back.

If time is short, preserve the valid old policy. It is safer to retry an update later than to leave the endpoint with corrupted or half-applied policy.

### 16. How do you isolate component failures?

Use **fault isolation** and **graceful degradation**.

- Separate privileged enforcement from networking and parsing when risk justifies it.
- Use bounded queues and worker pools.
- Add timeouts around network, filesystem, and IPC calls.
- Use circuit breakers for repeatedly failing dependencies.
- Keep telemetry overload from blocking enforcement.
- Apply CPU, memory, disk, and queue limits.
- Give components explicit health states: `HEALTHY`, `DEGRADED`, `FAILED`.
- Restart failed components independently where possible.

| Failure | Desired behavior |
|---|---|
| Communication | Retry; enter offline mode; enforcement continues |
| Telemetry | Drop or spool within a limit; never block enforcement |
| Policy update | Reject candidate; retain last-known-good policy |
| Privileged broker | Restart or enter safe mode; no unmediated fallback |
| Updater | Roll back; keep the current version |
| Corrupt state | Restore a valid snapshot or require re-enrollment |

The goal is not that every feature always works. The goal is that a non-critical failure does not disable protection, corrupt state, or create a security bypass.

---

## Cross-platform keywords

| Concern | Windows | Linux | macOS |
|---|---|---|---|
| Lifecycle | **SCM** | **systemd** | **launchd** |
| Identity | Service account, virtual account, per-service SID | UID/GID, groups, capabilities | UID/GID, entitlements, TCC, Keychain |
| Privileged IPC | Named pipes/RPC + ACLs | Unix socket + UID/GID/ACLs | XPC |
| Credentials | DPAPI/Credential Manager | Protected file/secret store | Keychain |
| Logging | Event Log | journald/syslog | Unified logging |
| Recovery | Failure actions | `Restart=`, watchdog, start limits | `KeepAlive` |

Keep shared interfaces platform-independent, but do not assume security semantics are identical across operating systems.

---

## Common interview traps

### “It polls the server every minute.”

Incomplete. Add **mTLS**, retry/backoff, durable queues, idempotency, policy versioning, and offline behavior.

### “Run it as root or LocalSystem.”

Incomplete. Start with **least privilege**, then isolate and justify only the operations that need elevation.

### “Restart it on crash.”

Incomplete. Add **backoff**, restart limits, health checks, crash diagnostics, safe mode, rollback, and alerting.

### “Store the token in a config file.”

Incomplete. Explain secure key generation, OS-protected storage, rotation, expiry, revocation, and log redaction.

### “Use the MAC address as the device ID.”

Weak. Use a device key pair and server-issued identity; hardware values are only metadata.

### “Kill the process during updates.”

Risky. Use **staging**, **signature verification**, **atomic activation**, **journaled state**, and **rollback**.

### “One process is always simpler.”

Discuss the trade-off between deployment simplicity and **security/fault isolation**. Split where a real boundary is needed.

---

## Final answer to memorize

> “I would build the endpoint agent as a headless, OS-supervised process with a shared cross-platform core and small OS-specific adapters. The communication layer would use mTLS, durable queues, retries, and backoff. The policy layer would validate and atomically activate versioned snapshots, while the enforcement layer would continue using the last-known-good policy during outages. I would isolate privileged operations behind a narrow authenticated IPC broker, store credentials in OS-protected storage, use signed staged updates with rollback, and recover from crashes with bounded restart and safe-mode behavior. The main goal is graceful degradation: the agent remains safe, observable, and locally useful even when the network or a non-critical component fails.”

