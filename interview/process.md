## 0–5 minutes: Introduction
### Tell us about yourself and your relevant experience? (100%) 2-3
I’m Quang, *a software engineer with two and a half years of experience* building distributed systems.
Previously, *At OPSWAT*, I worked on an intrusion detection system which is make of three components: Enterprise, Site, and Sensor. On the Sensor components, agents running on Windows and Linux that captured network traffic and sent it to Site. At Site, manage and modeling the data for asset, connection, policy, and vulnerability. Enterprise provided centralized management across multiple Sites.
I’m now at *AvePoint*, working on backend services for backup and migration data for a partner platform in the Microsoft Azure cloud ecosystem.
I’m interested in this role (SWE Desktop/Native) because it *algined with my experience* with background agents, distributed communication agents, networking, and security platform.

### “What interests you about Twin Signal and this Desktop/Native role?” **close to what I did**,**build further expertise in this area**
What draws me to Twin Signal is The Desktop/Native role is close to the work I've done before, where I  built and shipped a Windows/Linux app, worked on installer/update pipeline. That means I can contribute quickly without a long ramp-up.
At the same time, I'm looking to go deeper in this area. In my last role I worked on one Azure platform frequency, and this position would let me own more of native architecture, cross-platform behavior, performance, system-level integration. I'm excited to do that."

“Your recent work is mainly backend development. How does it prepare you for building endpoint agents?” **share several common aspects**, **valuable knowledge when bring it on**

---

## 5–15 minutes: Your experience and project ownership.

### “Walk us through the architecture of the product you worked on at OPSWAT. What did you personally own?”
"The product its an intrusion detection system for OT industrial. It had three tiers: Sensor, Site, and Enterprise. Data flowed upward from Sensor to Enterprise.
- Sensor is the data collection layer, At this layer I developed agents that capture packets and recognize Siemens, Schneider devices assets and its communication on specific network segments, packaged the installers for both platforms, WiX for Windows and .deb for Linux and distribute and mornitoring the service lifecycle.
- Site is the processing and management layer for one location. It receives normalized data from Sensors and builds the core model. I worked on manage devices and proflies of them. I also built connection visualization, showing relationships between assets as a graph. I worked on policy management and enforcement: when a policy was violated, Site could trigger a alert and call 3rd-party NAC, firewalls, and Aruba ClearPass to make an action, enrichment data integrate with ServiceNow or Miraki.
- Enterprise sits on top and provides centralized management across Sites: consolidated visibility, and configuration governance. I built parts of the dashboard for the cross-Site view. I worked on centralized configuration management, so settings could be pushed to multiple Sites from one place."
=> Possible asking how components comunication
"We used three mechanisms, chosen by the nature of the data:
- REST API for stateless request/response: configuration push, queries, dashboard data. Simple, cacheable, easy to retry.
- Sockets for low-latency, high-frequency signals: heartbeats and status. If one is lost, the next one replaces it, so occasional loss is acceptable.
- Message queue for critical data like alerts and asset events. Messages are persisted and acknowledged, so if the network drops between tiers, nothing is lost and the queue redelivers after reconnect. Because redelivery can cause duplicates, consumers dedupe with message IDs (idempotent processing)."

### “Tell us about a Windows service or background service you developed.”
"I developed the Sensor agent, a background service that runs on Windows and Debian Linux. It captures network traffic, normalizes the data, and sends it to our backend, Site, over an authenticated connection.
My part was wrote the service itself, handshake connection with site, simple extract and normalize data. On Windows I registered it with the Service Control Manager and handled start and stop requests. On Debian I wrote the systemd unit and set the service user and file permissions. I made sure that on a stop signal the agent closed its connections and released its resources cleanly, rather than getting killed mid-send.
I also built the resilience side. If the connection to Site dropped, the agent retried with backoff and buffered data locally and reconnected on its own. I configured recovery so that if the process crashed, the OS restarted it and it picked back up.
One thing I dealt with was [a real problem: e.g., permissions needed for packet capture, a service that hung on shutdown, a reconnect loop that hammered the backend, data lost during a restart] and I [what you did to fix it, plus the result]."
=> Possible asking detail to Windows Services/Daemons services

### “What was your involvement in MSI installers and software releases?”
I worked on the installers for our Windows and Linux agents from start to finish. I wrote the packaging, an MSI on Windows built with WiXToolset and a DEB on Debian, so that a fresh install laid down the agent, registered the service, and started it.
I also tested what I built. For every release I ran fresh installs and upgrades on clean machines/windows sandbox, checked that the service started, that it connected back to the backend, and that the existing config survived the update.
When something failed, I was the one debugging it. I'd go through the installer logs and service logs to find where it broke, then fix it in the package itself. For example, a config being overwritten on upgrade, a service not connect to Site after an update, a rollback that left things half-installed, and what you did about it.
On releases, I was part of the sign-off. I verified the build, confirmed test results, signed off on the installer side while the final call sat with devops. That all the scope I was involvement 


## 60+ minutes: Ref.

### “Describe a difficult production bug. How did you find the root cause and verify the fix?”
"We had a production issue where the connection between our components kept dropping, and it was hard to reproduce. Some devices would just stop sending data.
Our socket was supposed to exist only once in the whole app, but the dependency injection setup was quietly creating a second copy without any error. One copy kept the live connection, and the other was the one some parts of the code were actually using. When the wrong one lost its connection, everything depending on it went silent.
To find it, I looked at the logs around the drops and noticed the connection behaved as if it were two different objects. So I added logging in the constructor to see how many times it got created. It showed up twice, which confirmed the cause. Then I traced it back to how we registered it,it was registered in two different ways, so the system built one for each'.
I fixed the registration so only one instance exists, then checked it by running the same scenario that used to fail, and the constructor now logged only once. Nothing dropped over.
To prevent it from happening again, I added a test that checks the socket is created only once / added a startup check / documented how to register shared services."

### “Describe a deepfake project that you have worked on?“
"I worked on a research project about spotting deepfakes, mainly to protect face login on phones. The problem is that fake videos are getting very realistic, and tools that look for just one kind of clue often fail when the video is compressed or filmed with different cameras.
So we looked at two kinds of clues together. First, tiny traces that the fake-generation process leaves in the image, which you can't see by eye but show up when you analyze the image's patterns and noise. Second, how the person behaves: how they blink, where their eyes look, and how their head moves. Fake videos still struggle to get these natural human movements right.

### Question with no idea?“
Kubernetes (never used):
> "I haven't run Kubernetes in production. The closest thing I've done is packaging and managing service lifecycles with WiX and .deb installers. My understanding is that Kubernetes automates deployment, scaling, and restarts of containers. I'd start with a small local cluster like minikube to learn the core concepts, then deploy a simple service. Is that the kind of scenario you have in mind?"
Kafka (never used):
> "I haven't used Kafka directly. I've worked with RabbitMQ message queues for reliable delivery between tiers, so I understand acknowledgments and redelivery. My understanding is that Kafka is a distributed log built for high throughput and replay. I'd read up on partitions and consumer groups first, then prototype with a small topic."

# Windows Services & Linux Daemons

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

1. **What is a Windows Service?**  
A Windows Service is a background process managed by the **Service Control Manager (SCM)**. It can start automatically with Windows, run without a logged-in user, and be started, stopped, or restarted by the OS.

2. **What is a Linux daemon?**  
A Linux daemon is a long-running background process. On modern Linux systems, it is commonly managed by **systemd**, which controls startup, shutdown, restart behavior, dependencies, and logging.

3. **Windows Service vs Linux daemon?**  
They solve the same problem: running and managing background applications. Windows uses SCM, while Linux commonly uses systemd.

4. **What is a process?**  
A process is a running instance of a program. It has its own memory space, OS resources, security context, and one or more threads.

5. **Process vs thread?**  
A process provides isolation. A thread is an execution unit inside a process. Threads in the same process share memory, which makes communication fast but also creates synchronization problems such as race conditions.

6. **What happens when a service crashes?**  
The process terminates and the OS releases its process resources. SCM or systemd can detect the failure and restart the service based on its recovery configuration. The application itself should also be able to recover its previous state safely.

7. **How should a service shut down?**  
It should use graceful shutdown: stop accepting new work, finish or checkpoint current work, close connections, save important state, release resources, and then exit.

8. **What happens if the network connection is lost?**  
The service should normally keep running. Network failure should not mean service failure. The agent can retry the connection with backoff and synchronize again when connectivity returns.

9. **What is user mode vs kernel mode?**  
Applications normally run in **user mode**, where access is restricted. The operating system kernel runs in **kernel mode**, where it can access hardware and protected system resources. Applications request kernel operations through system calls.

10. **What does least privilege mean for a service?**  
A service should run with only the permissions it actually needs. Running everything as Administrator, root, or SYSTEM increases security risk. Privileged operations should be limited and isolated when possible.

The core mental model to remember is:

```text
Machine boots
    ↓
SCM / systemd
    ↓
Starts background service
    ↓
Process runs
    ↓
Threads execute work
    ↓
Service communicates with OS/network
    ↓
Handles failure gracefully
    ↓
SCM/systemd can restart it if necessary
```