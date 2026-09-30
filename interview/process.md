# 0–5 minutes: Introduction
## Tell us about yourself and your relevant experience? (100%)
> I’m Quang, *a software engineer with two and a half years of experience* building distributed systems. Previously, *At OPSWAT*, I worked on an ids which is make of 3 components: Enterprise, Site, and Sensor. On the Sensor, agents running on Windows and Linux that captured network traffic and sent it to Site. At Site, the data was modeling for asset, connection, and policy, and vulnerability. Enterprise provided centralized management across multiple Sites. I’m now at *AvePoint*, working on backup and migration data platform for Microsoft partner with Azure cloud ecosystem. I’m interested in this role (SWE Desktop/Native) because it *algined with my experience* with background agents, distributed communication agents, networking, and security platform.
---
## 5–10 minutes: Your experience and project ownership.
### Walk us through the product you worked on at OPSWAT. What did you personally own?
> The product its an intrusion detection system for OT industrial. It had three tiers: Sensor, Site, and Enterprise. Data flowed upward from Sensor to Enterprise. **(Sensor)** is the data collection layer, At this layer I developed full lifecyle agents that capture packets and recognize Siemens, Schneider devices and its communication on network segments, packaged the installers for both Win/Lin platforms. **(Site)** is the processing and management layer. It receives normalized data from Sensors and builds the core model. I worked on features manage devices and proflies of them. I also built connection visualization, showing relationships between assets as a graph. I also worked on policy management and enforcement: when a policy was violated, Site could trigger a alert and integrated 3rd-party NAC, firewalls, or Aruba ClearPass to make an action, enrichment data integrate with ServiceNow or Cisco Miraki. **(Enterprise)** sits on top and provides centralized management across Sites: visibility, and configuration. I built parts of the dashboard for the cross-Site view. I worked on centralized configuration management, so settings could be pushed to multiple Sites from one place."
---
# Required Qualifications
[Languages](#languages)
- Proficiency in at least one relevant systems/native language, such as Rust, Go, or a comparable language
- ~~(Preferred) Experience with JavaScript or Python (Typescript)~~
- ~~(Preferred) Experience with Rust or Go for systems-level or security-focused development~~
---
[Networking Foundations](#networking-foundations)
- Solid understanding of networking fundamentals: TCP/IP, DNS, routing, firewalls, VPN protocols, and/or ZTNA concepts.
- ~~(Preferred) Experience building agents/clients for RMM (remote monitoring and management), EDR/XDR, MDM, VPN, or ZTNA products.~~
---
[OS Services](#networking-foundations)
- Hands-on experience building, shipping, and maintaining background services, daemons, system agents, or other privileged/low-footprint desktop software.
- (Preferred) Experience with installers, silent deployment, code signing, auto-update systems, and large-scale software distribution 
- (Preferred) Experience developing agents for multiple desktop/server operating systems, including Windows, macOS, and Linux 
- Experience with OS-level programming concepts: services/daemons, permissions models, local user/group management, process management, and system APIs on Windows, macOS, and/or Linux.
---
[3rd party intergration](#3rd-party-integration)
- Experience designing and consuming REST APIs and integrating third-party systems, including identity providers and SaaS platforms. 
- Working knowledge of identity and access management concepts: authentication (SSO, SAML, OIDC), authorization models (RBAC/ABAC), and directory services. 
- ~~(Preferred) Direct experience integrating with identity providers (Entra ID/Azure AD, Okta, Ping, Google Workspace) or SaaS admin/security APIs (e.g., Google Workspace, Microsoft Graph, Slack, Salesforce)~~
---
[credentials](#credentials)
- Familiarity with local data storage, secure credential storage, encryption at rest/in transit, and secrets management. 
---
[communication](#communication)
- Familiarity with cloud platforms such as Azure or AWS, and real-time communication protocols (WebSockets, gRPC, MQTT) 
---
[scenarios](#scenarios)
- Ability to independently troubleshoot agent, network, identity-integration, and platform-specific issues
---
- ~~Strong professional English communication skills, both written and verbal. (practice)~~
- ~~3+ years of professional software engineering experience for mid-level candidates, or 5+ years for senior-level candidates.~~
- ~~Experience with Git, code review, automated testing, build tooling, and software release practices.~~
- ~~Excellent analytical, organizational, prioritization, documentation, and collaboration skills~~
- ~~Experience with Docker, CI/CD pipelines, infrastructure-as-code, or deployment automation~~
- ~~(Preferred) Knowledge of secure software development, threat modeling, encryption, secrets management, and data privacy/compliance practices (e.g., SOC 2, ISO 27001).~~ 
- ~~(Preferred) Experience with policy engines, rules evaluation systems, or attribute-based access control implementations~~
- ~~(Preferred) Experience working in a security product, consulting, or distributed-team environment~~ 
- ~~(Preferred) Prior experience mentoring developers or leading small technical initiatives~~ 
--- 
## 15–30 minutes:
### Tell us about a Windows service or Linux/Daemons background service you developed.

Windows services are background processes that run independently of user login, managed by the Service Control Manager (SCM).

**Core concepts:**
- **SCM (Service Control Manager)** — `services.exe`, runs at boot, starts/stops/monitors all services
- **Service executable** — implements `ServiceMain()` and a control handler to respond to start/stop/pause requests
- **Startup types**:
  - Automatic — starts at boot
  - Automatic (Delayed Start) — starts shortly after boot, reduces startup contention
  - Manual — starts on demand
  - Disabled — can't be started
- **Service accounts** — context a service runs under:
  - `LocalSystem` — highest privilege, full OS access
  - `LocalService` — minimal privileges, network access as anonymous
  - `NetworkService` — minimal privileges, network access as machine account
  - Custom user/domain account — for specific permission needs
- **Service SIDs / isolation** — each service can get its own SID so its resources can be locked down independently of others sharing a host process
- **Session 0 isolation** (since Vista) — services run in Session 0, separate from user sessions, so they can't interact with the desktop

**Service hosting:**
- Standalone `.exe` per service, or
- Shared `svchost.exe` processes — many Windows services are DLLs grouped into shared host processes to reduce overhead

**Management tools:**
- `services.msc` — GUI
- `sc.exe` — command-line (create, config, query, start, stop)
- `PowerShell` — `Get-Service`, `Start-Service`, `Stop-Service`, `Set-Service`, `New-Service`
- Registry — service configs live under `HKLM\SYSTEM\CurrentControlSet\Services`

**Lifecycle/state machine:**
`Stopped → Start Pending → Running → Stop Pending → Stopped` (plus Pause/Continue states if supported)

**Dependencies:** services can declare dependencies on other services or drivers; SCM starts them in the correct order.

**Recovery options:** each service can define actions on failure (restart service, run a program, reboot machine), configurable per failure count.

**Security-relevant notes:**
- Weak service permissions (e.g., writable binary path, unquoted service paths) are a classic privilege-escalation vector
- Services running as `LocalSystem` are high-value targets if compromised

Here's a deeper look at the registry structure and runtime behavior of Windows services.

**Registry location**

All services (and drivers) are registered under:
```
HKLM\SYSTEM\CurrentControlSet\Services\<ServiceName>
```

Key values under each service key:
- **ImagePath** — path to the executable (or driver .sys file)
- **Start** — startup type:
  - `0` = Boot (driver, loaded by kernel loader)
  - `1` = System (driver, loaded by I/O subsystem)
  - `2` = Automatic
  - `3` = Manual
  - `4` = Disabled
- **Type** — what kind of service:
  - `1` = Kernel driver
  - `2` = File system driver
  - `0x10` = Own process (`SERVICE_WIN32_OWN_PROCESS`)
  - `0x20` = Shared process (`SERVICE_WIN32_SHARE_PROCESS`, i.e. svchost-hosted)
  - `0x110`/`0x120` = interactive variants (rare, legacy)
- **ErrorControl** — what happens if the service fails to start (ignore/normal/severe/critical, affects boot behavior)
- **ObjectName** — account the service runs as (e.g. `LocalSystem`, `NT AUTHORITY\NetworkService`, or a domain user)
- **DependOnService** / **DependOnGroup** — dependency list
- **DisplayName**, **Description**
- **FailureActions** (binary blob) — recovery settings (restart/run command/reboot, reset period)

For svchost-hosted services, there's also a `Parameters` subkey with `ServiceDll` pointing to the DLL, since the actual code isn't in an executable at ImagePath — svchost.exe is the ImagePath, and it loads the DLL.

**Runtime / boot sequence**

1. Bootloader loads kernel + boot-start drivers (`Start=0`) directly.
2. Kernel finishes init, I/O manager loads system-start drivers (`Start=1`).
3. `wininit.exe` starts `services.exe` (the SCM).
4. SCM reads the registry, builds a dependency graph, and starts:
   - Auto-start services in dependency order
   - Delayed-auto-start services shortly after (via a separate timer, off the critical boot path)
5. Manual-start services wait for something to request `StartService()` (another service, an app, or a user via `services.msc`/`sc start`).
6. Each service process calls `StartServiceCtrlDispatcher()` early in `main()`, registering a `ServiceMain` entry point per service name — this is how one .exe can host multiple services (like `svchost.exe`).
7. `ServiceMain` calls `RegisterServiceCtrlHandlerEx()` to receive control codes (stop, pause, shutdown, custom), then reports `SERVICE_RUNNING` via `SetServiceStatus()`.
8. SCM polls/expects periodic status updates during pending states (`dwWaitHint`, `dwCheckPoint`) — if a service doesn't respond in time, SCM considers it hung.

**Useful runtime inspection commands**
```
sc query <name>            # current state
sc qc <name>                # query config (ImagePath, start type, account)
sc queryex <name>           # includes PID
reg query HKLM\SYSTEM\CurrentControlSet\Services\<name>
tasklist /svc               # map PIDs to hosted services
Get-Service | ? Status -eq Running
```
---
Linux equivalents: daemons are background processes, and **systemd** is the init system/service manager on most modern distros (replacing SysV init and Upstart).
**Daemon fundamentals**
- A daemon is a long-running background process, typically with no controlling terminal, named with a trailing `d` (`sshd`, `crond`, `systemd-journald`).
- **Classic (SysV-style) daemonization**: fork, `setsid()` to detach from the terminal, fork again (so it can't reacquire a TTY), `chdir("/")`, reset `umask`, close/redirect stdin/stdout/stderr to `/dev/null`, write a PID file.
- **Modern approach**: don't daemonize yourself. Run in the foreground and let systemd supervise (`Type=simple`), logging to stdout/stderr (captured by journald).

**Init history**
- **SysV init**: shell scripts in `/etc/init.d/`, runlevels 0-6, symlinks in `/etc/rc*.d/`, sequential startup.
- **Upstart**: event-driven (Ubuntu, briefly).
- **systemd**: parallel startup, dependency-based, socket/bus/timer activation, cgroup tracking. PID 1.

**systemd core concepts**
- **Units** are the objects systemd manages. Types:
  - `.service` — a daemon/process
  - `.socket` — socket activation
  - `.timer` — cron-like scheduling
  - `.target` — grouping/sync points (like runlevels)
  - `.mount`, `.automount`, `.device`, `.path`, `.slice`, `.scope`, `.swap`
- **Targets**: `multi-user.target` (~runlevel 3), `graphical.target` (~5), `rescue.target`, `default.target` (symlink to the boot target).
- **cgroups**: each service runs in its own cgroup, so systemd can track all child processes and kill them reliably on stop.

**Unit file locations (precedence high to low)**
```
/etc/systemd/system/          # admin-created/overrides
/run/systemd/system/          # runtime
/usr/lib/systemd/system/      # package-installed (don't edit)
~/.config/systemd/user/       # per-user units
```
Use drop-ins (`systemctl edit foo.service` → `/etc/systemd/system/foo.service.d/override.conf`) rather than editing packaged files.

**Example service unit**
```ini
[Unit]
Description=My App
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp --config /etc/myapp.conf
User=myapp
Group=myapp
Restart=on-failure
RestartSec=5
Environment=LOG_LEVEL=info

[Install]
WantedBy=multi-user.target
```

**Key `[Service]` options**
- **Type**: `simple` (default), `exec`, `forking` (classic daemon, needs `PIDFile=`), `oneshot`, `notify` (service signals readiness via `sd_notify`), `dbus`, `idle`
- **Restart**: `no`, `on-failure`, `always`, `on-abnormal`, etc.
- **ExecStartPre / ExecStartPost / ExecStop / ExecReload**
- **TimeoutStartSec / TimeoutStopSec**
- **User / Group / DynamicUser**

**Dependencies and ordering** (separate concepts):
- `Requires=`, `Wants=`, `BindsTo=`, `Conflicts=` — *what* gets pulled in
- `After=`, `Before=` — *order* only

**Hardening options** (a big systemd strength)
```
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
RestrictAddressFamilies=AF_INET AF_INET6
MemoryMax=512M
```
Check the score with `systemd-analyze security myapp.service`.

**Management commands**
```
systemctl start|stop|restart|reload|status <unit>
systemctl enable|disable <unit>      # create/remove WantedBy symlinks
systemctl enable --now <unit>
systemctl mask|unmask <unit>         # like "Disabled" but stronger (links to /dev/null)
systemctl daemon-reload              # after editing unit files
systemctl list-units --type=service
systemctl list-unit-files
systemctl cat|show|edit <unit>
systemctl --user ...                 # per-user manager
systemctl get-default / set-default multi-user.target
```

**Logging: journald**
```
journalctl -u myapp.service -f       # follow
journalctl -b                        # this boot
journalctl -p err --since "1 hour ago"
journalctl -xe                       # recent errors with explanations
```

**Boot analysis**
```
systemd-analyze                      # total boot time
systemd-analyze blame                # slowest units
systemd-analyze critical-chain
```

**Timers (cron replacement)**
```ini
# backup.timer
[Timer]
OnCalendar=daily
Persistent=true
[Install]
WantedBy=timers.target
```
Paired with `backup.service`.

**Socket activation**: systemd listens on a port/socket and starts the service on first connection, passing the file descriptor. Enables on-demand start and zero-downtime restarts.

**Windows to Linux mapping**

| Windows | Linux (systemd) |
|---|---|
| SCM (`services.exe`) | systemd (PID 1) |
| Service | `.service` unit |
| Registry `Services\<name>` | Unit files in `/etc/systemd/system` |
| Startup type Automatic | `enable` (WantedBy target) |
| Manual / Disabled | not enabled / `mask` |
| `LocalSystem` / custom account | `User=` (root default) / `DynamicUser=` |
| Service SID isolation | Sandboxing options (`ProtectSystem`, `PrivateTmp`, etc.) |
| Recovery options | `Restart=`, `OnFailure=` |
| Dependencies | `Requires=`/`Wants=` + `After=` |
| `sc` / `Get-Service` | `systemctl` |
| Event Log | journald / `journalctl` |
| Scheduled Tasks | `.timer` units / cron |
| Session 0 isolation | Services have no session/TTY by default |

**Security notes**
- Writable unit files or writable `ExecStart` binaries owned by non-root are a privilege-escalation vector (analogous to weak service permissions on Windows).
- Services running as root without sandboxing are high-value targets; prefer unprivileged users plus hardening directives.

---

## communication
### How would you choose how components communicate?
> We used three mechanisms, chosen by the nature of the data: REST API for stateless request/response: configuration push, queries, dashboard data. Simple, cacheable, easy to retry. Sockets for low-latency, high-frequency signals: heartbeats and status. If one is lost, the next one replaces it, so occasional loss is acceptable. Message queue for critical data like alerts and asset events. Messages are persisted and acknowledged, so if the network drops between tiers, nothing is lost and the queue redelivers after reconnect. Because redelivery can cause duplicates, consumers dedupe with message IDs (idempotent processing).
---

## Languages
### Your strongest language is C#. How comfortable are you with Rust or Go?
> C# is my strongest language, but I don't see switching languages as a big obstacle. The core concepts carry over: types, concurrency, memory, error handling, and API design. What changes is the idioms, like goroutines in Go or ownership in Rust. I also use AI to explain unfamiliar idioms and translate C# patterns into idiomatic Go or Rust, and I always check its output against the docs and code review so I'm actually working with the language.
---

## Networking Foundations
| Topic | Interview-ready understanding |
|---|---|
| **TCP vs UDP** | Both are **transport-layer protocols**. **TCP** provides a reliable, ordered datagrams: it establishes a connection, tracks sequence numbers, acknowledges data, and retransmits lost data. **UDP** sends independent datagrams with no guarantee of delivery, tracks ordering, or retransmission, trading reliability for lower overhead and latency. |
| **DNS** | DNS translates human-friendly domain names such as `google.com` into information computers can use, most commonly IP addresses such as `142.x.x.x`. |
| **Routing** | Network routing decides **which path packets take between networks**, based primarily on destination IP addresses and routing tables. |
| **Ports** | An IP identifies a **machine/network interface**, while a port identifies a particular network service/process endpoint on that machine. For example `192.168.1.10:443`: IP → machine, port `443` → service listening there. |
| **Sockets** | A socket is the **programming abstraction/API** applications use to communicate over the network. You can create a TCP socket or UDP socket. A network connection is commonly identified by protocol + source IP/port + destination IP/port. |
| **HTTP** | HTTP is an **application-layer request/response protocol**. HTTP/1.1 and HTTP/2 normally run over TCP. HTTP itself is stateless: each request contains the information needed to process it, although applications can maintain state using cookies, tokens, sessions, databases, etc. HTTP/3 is different: it runs over QUIC, which uses UDP. |
| **WebSocket** | WebSocket gives the client and server a **persistent, full-duplex connection**, allowing either side to send messages at any time. Important correction: traditional WebSocket normally runs over **TCP, not UDP**. It usually begins with an HTTP handshake and then upgrades the connection to WebSocket. |
| **Firewall** | A firewall enforces rules controlling network traffic. Rules can consider things like source/destination IP, ports, protocol, connection state, application, interface, etc., and decide whether traffic is allowed or blocked. |
| **VPN** | It creates an encrypted tunnel between your device and another network/VPN gateway. It can route some or all network traffic through that tunnel, making your device logically connected to the remote/private network. |
| **ZTNA** | Zero Trust Network Access provides access based on **identity, device posture, policy, and context**, rather than trusting someone merely because they are connected to the corporate network. Typically, it grants access to specific applications/resources instead of giving broad network access like a traditional VPN. |

### VPN vs ZTNA

So a strong interview answer would be:
> A traditional VPN establishes an encrypted tunnel and usually gives the device network-level access to a private network. ZTNA follows zero-trust principles: it continuously evaluates identity, device posture, and policy, and grants access to specific resources rather than implicitly trusting a device because it's inside the network.
---

## 3rd party integration
> I have solid experience designing and consuming REST APIs, and I’ve integrated with several third-party systems.
In my previous work, I integrated with platforms and SDKs such as ServiceNow, Cisco Meraki, Aruba ClearPass, Siemens, Schneider Electric, and Rockwell Automation. Depending on the integration, I worked with REST APIs or vendor SDKs, handled authentication, mapped external data into our internal models, processed errors and timeouts, and made sure the integration could recover when the external system was temporarily unavailable.
I haven’t directly implemented an identity-provider integration such as Okta or Entra ID yet. However, I’m familiar with the anothers third party integration so I’m confident I could pick up that part quickly.

---

## Credentials
Mục này họ thường **không kỳ vọng bạn là security engineer chuyên cryptography**. Với role endpoint/agent, họ muốn biết bạn có tư duy đúng về việc **agent lưu dữ liệu local và giữ secret an toàn**.

Cụ thể họ thường expect bạn hiểu 4 phần:
Local data storage
- Agent của bạn lưu dữ liệu local ở đâu? Sqlite Datbase + cypher encripted + DPAPI (windows encrypted cyperkey) + systemd-creds (Linux).
- Nếu network/backend unavailable thì bạn xử lý buffered data thế nào?
- Làm sao tránh corruption khi service crash/restart?
- Files permissions ACLs for Windows and chmod chown for Linux (600, 750, 755)

Secure credential storage
- Bạn sẽ lưu API token/password/client secret của agent ở đâu? Sqlite Datbase + cypher encripted + DPAPI (windows encrypted cyperkey) + systemd-creds (Linux).
- Trên Windows bạn biết cơ chế nào để bảo vệ credential? Expected: Windows DPAPI, ACL.
- Linux thì sao? Expected: restrictive file permissions (chmod, chown), secret stores/keyrings nếu phù hợp.
- Nếu attacker copy được config file sang máy khác thì họ có sử dụng secret đó được không?

Encryption at rest / in transit
Encryption in transit protects data while it is travelling between systems, usually using TLS, such as HTTPS between an agent and backend.  
Encryption at rest protects stored data, for example a local database, cached sensitive information, or credentials stored on disk.  
They solve different problems, so normally sensitive systems need both.

Secrets management
- Azure Key Vault, Vault, Dev just consume through API, this handle by Devops 

## Scenarios
### How would you investigate high CPU or memory usage on a customer’s device?
> I'd start by finding out which process, version, and devices are affected, and when it began. On the device, I'd use `htop` on Linux or Task Manager on Windows to see what's using CPU or RAM and whether it keeps growing. I'd also add a simple health check to the agent that reports its own CPU and RAM every minute to our monitoring, with an alert if it stays above a threshold, and let systemd or the Windows service settings restart it if it crashes. Then I'd check the logs around when the problem began, and if needed, take memory dumps to see what's growing. To reproduce it, I'd run the same version and config in a test environment and leave it running for a few hours while watching CPU and memory. After fixing it, I'd add an alert so we catch it earlier next time.

### How would you keep an agent reliable over a long time?
> I would avoid busy loops, limit concurrent work and queue sizes, and release resources when they are no longer needed. I would add useful logs and monitor memory, CPU, and recent successful activity. Expected failures, such as a temporary network problem, should be handled. The service manager can restart a crashed process, but a process that is alive and stuck needs a separate health check.

### Describe a difficult production bug. How did you find the root cause and verify the fix?
> We had a production issue where the connection between our components kept dropping, and it was hard to reproduce. Some devices would just stop sending data.
Our socket was supposed to exist only once in the whole app, but the dependency injection setup was quietly creating a second copy without any error. One copy kept the live connection, and the other was the one some parts of the code were actually using. When the wrong one lost its connection, everything depending on it went silent.
To find it, I looked at the logs around the drops and noticed the connection behaved as if it were two different objects. So I added logging in the constructor to see how many times it got created. It showed up twice, which confirmed the cause. Then I traced it back to how we registered it,it was registered in two different ways, so the system built one for each'.
I fixed the registration so only one instance exists, then checked it by running the same scenario that used to fail, and the constructor now logged only once. Nothing dropped over.
To prevent it from happening again, I added a test that checks the socket is created only once / added a startup check / documented how to register shared services.

<!--
### 2. What is different about a Windows service or Linux daemon?

> It runs in the background and can run without an interactive user logging in. On Windows, the Service Control Manager manages the service. On Debian, systemd can manage it. It still runs under an account with defined permissions. I would configure startup and recovery, and make sure the application handles shutdown properly. Running in the background does not mean it has unrestricted access.

**Remember:** Background process → service manager → account and permissions.

### 3. What permissions should the service use?

> I would first identify the operations it needs, then give its account only the required permissions. On Windows, LocalSystem has extensive privileges, so I would not choose it automatically. On Linux, I would prefer a dedicated service user where possible. File ownership and permissions control what that user can access. If an operation needs extra privileges, I would check the specific OS or driver requirements.

**Follow-up:** `chown` changes ownership and `chmod` changes file permissions; neither chooses the account that systemd runs the service as. That is configured separately, for example with `User=`.

### 4. How would you handle startup, dependencies, and shutdown?

> At startup, I would validate configuration and initialize the resources the service needs. Startup ordering helps with local dependencies, but it does not guarantee a remote backend is ready. I would handle that with connection retries. During shutdown, I would stop accepting new work, signal cancellation, give current work a limited time to finish, and close connections and files.

**Remember:** Initialize → handle unavailable dependencies → cancel and clean up.

**Remember:** Bounded work → resource cleanup → health checks → recovery.

### 6. What should happen when the backend is unavailable?

> I would use timeouts and retry temporary failures with increasing delays, up to a maximum delay. Some randomness in the delay helps devices avoid reconnecting together. If data must survive an outage, I would use a bounded local buffer and define what happens when it fills. After reconnecting, I would send pending work carefully. Authentication or invalid-request errors need investigation rather than repeated retries.

**Remember:** Timeout → backoff → bounded buffer → reconnect.

### 7. How would you prevent the same command from being applied twice?

> I would give each command an ID and keep a durable record of its status. I would also try to make the action safe to repeat, such as setting a configuration value to the required value. There is still a difficult case if the process crashes after changing the OS but before recording success. On restart, I would check the actual state before repeating the action.

**Remember:** Command ID + durable status + check actual state.

**Honest limit:** A list of completed IDs alone does not guarantee exactly-once execution across a crash.

### 8. What is the difference between a process, a thread, and async/await?

> A process has its own address space. Threads run inside a process and share its memory, so shared data needs care. Async/await lets code wait for operations such as network I/O without blocking a thread during the wait. It does not automatically create another thread or make CPU-heavy work faster. For CPU-heavy work, I would consider parallel execution and limit concurrency.

**Remember:** Process → isolation; threads → shared memory; async → efficient waiting.

### 9. What are race conditions and deadlocks?

> A race condition happens when the result depends on the timing of concurrent operations on shared state. For example, two workers might both see a command as pending and execute it. A deadlock happens when operations wait on each other and cannot proceed. I would reduce shared mutable state, use appropriate synchronization, keep locked sections small, and use a consistent order when acquiring multiple locks.

**C# follow-up:** Avoid blocking async work with `.Result` or `.Wait()`. Use a suitable async synchronization mechanism when an operation needs to await while holding exclusive access.

### 10. The agent cannot connect to the server. How would you investigate?

> I would first check the error, the affected devices, and any recent changes. Then I would check the configured address, DNS resolution, connectivity to the actual server port, and any firewall or proxy restrictions. If the connection succeeds, I would check TLS and authentication errors, then the server logs. I would work through these steps rather than changing several settings at once.

**Remember:** Address → DNS → port/connectivity → TLS → authentication → application.

---

## 30–45 minutes: Agent design and security scenarios

**Practice notes:** Explain a small, understandable design and its failure cases. You do not need to claim you have designed a complete endpoint security product. Ask about the target OS, the actions the agent performs, and the required offline behavior before choosing a detailed design.

### 1. How would you design an agent that receives and applies policies?

> I would separate communication with the backend, policy validation, and OS-specific actions. The agent would receive a versioned policy, validate it, compare it with the current state, and apply the required changes. It would then report what succeeded or failed. I would start with one supported OS and one policy type, test failure cases, and extend it from there.

**Remember:** Receive → validate → compare → apply → report.

### 2. How would you check whether the policy was actually applied?

> I would check the resulting OS state rather than assuming a successful command means everything is correct. The agent would report the policy version and the result of each action. If only part of the policy succeeded, it should report partial failure. I would also check periodically for changes made outside the agent. Whether to reapply them automatically would depend on the agreed policy.

**Remember:** Desired state → actual state → report differences.

### 3. What if the agent needs administrator privileges?

> I would identify which actions need elevated access and keep that access as limited as practical. Commands should have a defined format and allowed actions, rather than accepting arbitrary scripts or shell commands. I would validate inputs and restrict who can call privileged operations. Before shipping a new privileged feature, I would ask for a security review and test it in an isolated environment.

**Follow-up:** If asked about separating privileged operations into another process, explain the idea and say if you have not implemented it. Do not present it as previous experience.


**If asked for an unfamiliar OS API:** Say you would check the supported mechanism for that service environment rather than guess a function name.

### 5. How would you make automatic updates safer?

> I would verify that the update comes from a trusted publisher and is intended for this platform and version before running it. I would preserve configuration, stop the service cleanly when required, install the update, and check that the service works afterward. I would also plan recovery if the update fails. I would not assume every installer automatically rolls everything back, especially if stored data has changed.

**Remember:** Verify → preserve → install → health check → recover.

**Experience boundary:** Explain your actual MSI/WiX or DEB packaging work when asked. Discuss code signing, rollout control, or automatic rollback as proposed steps unless you personally implemented them.

### 6. What is authentication versus authorization? What about RBAC and ABAC?

> Authentication checks who the user or device is. Authorization checks what it is allowed to do. RBAC assigns permissions through roles, such as administrator or viewer. ABAC uses attributes, such as department, device status, or time, to make an access decision. For an agent, a valid device identity alone should not authorize every command; the requested action must also be allowed.

**Remember:** Who are you? → What may you do? → Roles or attributes.

**Remember:** VPN → protected network connection; ZTNA → policy-based access to resources.

**Experience boundary:** State any actual integration you have done separately; knowing these concepts does not mean you have built a VPN or ZTNA client.

### 8. A firewall policy blocks the agent's own connection. What would you do?

> First, I would confirm whether the new rule caused the failure and use the approved local or out-of-band recovery path if remote access is lost. To reduce the risk, I would test policies on a small group and validate the required management connection. Where supported, I would use a time-limited change that reverts if connectivity cannot be confirmed. That recovery needs to exist before the failure happens.

**Remember:** Preserve management access → test → verify → planned recovery.

### 9. Should the agent allow or deny access when the server is unavailable?

> I would clarify the product's security requirements first. Continuing access may preserve availability, while denying it may reduce the risk of unauthorized access. A product might permit some operations using a valid cached policy and block others. I would define which actions are allowed offline, how long the policy remains valid, and what happens when it expires, with the security team.

**Remember:** Agree offline rules before an outage; there is no universal default for every action.

### 10. What would you do if you did not know a security or platform detail?

> I would explain the part I understand and be clear about the part I have not implemented. Then I would check the existing code and official documentation, build a small test, and ask for review where the change affects security or OS behavior. I would rather verify the behavior than make an assumption that could affect a customer's device.

**Speaking pattern:** “My understanding is … The part I would need to verify is …”

### Technical references for these added sections

- [Microsoft: Retry pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry) — transient failures, retry delays, and idempotency.
- [Microsoft: Service accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts) — service identities and account selection.

----->

## 60+ minutes
### Describe a deepfake project that you have worked on?
"I worked on a research project about spotting deepfakes, mainly to protect face login on phones. The problem is that fake videos are getting very realistic, and tools that look for just one kind of clue often fail when the video is compressed or filmed with different cameras.
So we looked at two kinds of clues together. First, tiny traces that the fake-generation process leaves in the image, which you can't see by eye but show up when you analyze the image's patterns and noise. Second, how the person behaves: how they blink, where their eyes look, and how their head moves. Fake videos still struggle to get these natural human movements right.
### What are your availability, notice period, and expectations for remote collaboration?
My notice period is 30 days, starting from the day I accept the offer, so I could start right after that. I'm comfortable working remotely and I'm used to collaborating that way.
### Question with no idea?
Kubernetes (never used):
> "I haven't run Kubernetes in production. The closest thing I've done is packaging and managing service lifecycles with WiX and .deb installers. My understanding is that Kubernetes automates deployment, scaling, and restarts of containers. I'd start with a small local cluster like minikube to learn the core concepts, then deploy a simple service. Is that the kind of scenario you have in mind?"

Kafka (never used):
> "I haven't used Kafka directly. I've worked with RabbitMQ message queues for reliable delivery between tiers, so I understand acknowledgments and redelivery. My understanding is that Kafka is a distributed log built for high throughput and replay. I'd read up on partitions and consumer groups first, then prototype with a small topic."
