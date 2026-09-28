# Windows Services, Linux Daemons, and Cross-Platform OS Integration

Interview preparation for a **Software Engineer with ~3 years of experience**.

## The core mental model

A normal application is usually started and stopped by a user. A **service/daemon** is a long-lived process whose **lifecycle**, **readiness**, **identity**, **permissions**, **logging**, and **failure recovery** are managed by the operating system.

The most useful interview framing is:

> “The process contains the application logic; the service manager owns the process lifecycle and provides a contract for starting, stopping, monitoring, and recovering it.”

The names differ by platform:

| Platform | Process manager | Definition | Typical control interface |
|---|---|---|---|
| Windows | **Service Control Manager (SCM)** | Service registration in the **Service Control Manager database**, backed by the Registry | `services.msc`, `sc.exe`, PowerShell |
| Linux | Usually **systemd** | A **unit file** describing the service and its dependencies | `systemctl`, `journalctl` |
| macOS | **launchd** | A **property list (plist)** describing a daemon or agent | `launchctl`, `SMAppService` |

---

## Windows Services

### What is a Windows Service, and how is it different from a normal Windows process?

A **Windows Service** is a background program registered with the **Service Control Manager (SCM)**. It is designed to run without an interactive user, often start during boot, run under a controlled identity, and respond to lifecycle commands such as start, stop, pause, continue, and shutdown.

A service is still a **Windows process**. The important difference is the contract around that process:

| Normal process | Windows Service |
|---|---|
| Usually launched by a user, shell, scheduler, or another process | Started and controlled by **SCM** |
| May own a visible window or console | Normally has **no UI** and runs in a non-interactive session |
| Exit is generally just an application exit | Must report **service status** and handle control requests |
| No standard recovery behavior | Can have **automatic recovery actions** |
| Identity is chosen by the launcher/user | Runs under a configured **service account** |

An executable does not become a service merely because it is a background process. It must either implement the **service entry-point contract** or be hosted by a service wrapper that does.

**Keywords interviewers want to hear:** **SCM**, **service lifecycle**, **service status**, **non-interactive**, **service account**, **recovery actions**, **not just a background process**.

### How does the Windows Service Control Manager (SCM) work?

The SCM is the Windows component that maintains the service database and coordinates service lifecycles. Its main responsibilities are:

1. Read a service's configuration: executable path, startup type, account, dependencies, and recovery policy.
2. Start services according to their **startup type** and **dependency graph**.
3. Launch the service process and connect it to SCM through the service-dispatch mechanism.
4. Accept control requests such as `START`, `STOP`, `PAUSE`, `CONTINUE`, and `SHUTDOWN`.
5. Track reported states such as **START_PENDING**, **RUNNING**, **STOP_PENDING**, and **STOPPED**.
6. Enforce timeouts and detect service failures.
7. Apply configured **failure actions**, such as restarting the service.

The SCM itself runs as the `services.exe` process. Administration tools such as `services.msc`, `sc.exe`, PowerShell, and management APIs communicate with SCM rather than directly manipulating the service process.

SCM also handles **dependencies**. If service A depends on service B, SCM starts B first and normally stops A before B. Dependency declarations are not the same as ordering-only relationships: a dependency expresses a required relationship, while ordering expresses sequence.

**Good interview answer:**

> “SCM is the lifecycle owner. It reads the service registration, resolves dependencies, launches the configured executable under the configured identity, receives service status updates, sends control notifications, and applies timeout and recovery policy.”

### What happens internally when Windows starts a service?

The exact details vary by boot phase and service type, but the normal flow is:

1. During boot, Windows starts **SCM**.
2. SCM reads the service configuration and identifies services whose `Start` value is **Automatic**. It also considers **delayed start**, trigger-start rules, dependencies, and service groups.
3. SCM creates or launches the service process using the configured **ImagePath** and **ObjectName** account.
4. The process calls `StartServiceCtrlDispatcher`, connecting the process to SCM. This is why a service executable must not behave like an ordinary console application when launched by SCM.
5. SCM invokes the service's service-main function.
6. The service registers a **control handler** with SCM and reports **START_PENDING** while it initializes. It should provide a sensible **wait hint** and increment its **checkpoint** if initialization takes time.
7. Once ready to accept work, the service reports **RUNNING** and indicates which controls it accepts.
8. SCM considers startup successful and continues with other services whose dependencies are now satisfied.

If initialization fails, the service should report **STOPPED** with an appropriate exit code. If it never reports a valid state or exceeds the startup timeout, SCM records a failure and may invoke the configured recovery action.

**Important distinction:** process creation is not the same as service readiness. A process can exist while the service is still **START_PENDING** and not yet ready to serve requests.

### Where does Windows store service registration information in the Registry?

The main location is:

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\<ServiceName>
```

For example:

```text
HKLM\SYSTEM\CurrentControlSet\Services\MyAgent
```

Common values include:

| Registry value | Meaning |
|---|---|
| `ImagePath` | Executable path and command-line arguments |
| `Type` | Service type, such as own-process or shared-process |
| `Start` | Startup type: boot, system, automatic, manual, or disabled |
| `ErrorControl` | What boot should do if a service fails |
| `ObjectName` | Account under which the service runs |
| `DependOnService` | Services that must be started first |
| `DependOnGroup` | Required service groups |
| `DelayedAutoStart` | Whether an automatic service uses delayed start |
| `FailureActions` | Binary configuration for service recovery actions |
| `FailureActionsOnNonCrashFailures` | Whether certain non-crash failures trigger recovery |

`CurrentControlSet` is the active control-set alias. Windows may maintain other control sets, such as `ControlSet001`, but service code should normally use the active alias rather than hard-coding a control-set number.

**Security note:** Do not treat the Registry as a safe place to store application secrets. Service credentials are managed by Windows and should not be placed in ordinary service configuration values.

### What is the purpose of `HKLM\SYSTEM\CurrentControlSet\Services`?

It is the Registry-backed configuration area for the **SCM service database**. Each child key describes one service or driver and tells Windows:

- what executable or driver to load;
- when to load it;
- which account should run it;
- what it depends on;
- which controls and recovery behavior apply.

The key is configuration, not the service's runtime state. Runtime status is obtained through SCM APIs or tools such as `sc.exe query`; changing a Registry value does not itself start or stop a running service and can be unsafe if done incorrectly.

Prefer supported management APIs, `sc.exe`, PowerShell, or an installer framework for registration and changes.

### Windows service startup types

| Startup type | Meaning | Typical use |
|---|---|---|
| **Automatic** | SCM starts the service during system startup, subject to dependencies | Core infrastructure needed soon after boot |
| **Automatic (Delayed Start)** | SCM starts it automatically after the initial automatic-start workload | Non-critical services that should start after boot pressure decreases |
| **Manual** | SCM does not start it at boot; it starts on demand, through an explicit request or a configured trigger | Utilities, optional components, trigger-start services |
| **Disabled** | The service cannot be started until an administrator changes the configuration | Intentionally unavailable or decommissioned services |

The startup type answers **when SCM may start the service**. It does not determine the service account, permissions, readiness behavior, or recovery policy.

A modern service may also use **service triggers**, such as a device arrival, network availability, or a named event. Therefore “Manual” does not always mean “only a human clicked Start.”

### LocalSystem, LocalService, NetworkService, and a dedicated service account

The account determines the service's **security token**, local permissions, network identity, and audit boundary.

| Account | Local privileges | Network identity | Appropriate use |
|---|---|---|---|
| **LocalSystem** (`NT AUTHORITY\SYSTEM`) | Very powerful local identity; broad access to the machine | Usually the **computer account** on the network | Only when the service truly needs extensive local privileges |
| **LocalService** (`NT AUTHORITY\LOCAL SERVICE`) | Low local privileges | Usually presents as anonymous or with limited identity | Services needing minimal local access and little network identity |
| **NetworkService** (`NT AUTHORITY\NETWORK SERVICE`) | Low local privileges, with some built-in rights | Presents the machine's computer account to remote resources | Services needing low local privilege plus authenticated machine access |
| **Dedicated service account** | Explicitly assigned least-privilege rights | Can have a controlled user/domain identity | Production services needing a distinct audit and access boundary |

For a dedicated account, prefer a **managed service account** or **group Managed Service Account (gMSA)** where supported. It reduces password handling and allows centralized rotation. Grant only the required **Log on as a service**, filesystem, registry, network, and database permissions.

Also consider a **virtual service account** or **per-service SID** when it provides a narrower identity without creating a traditional user account.

**Best-practice answer:**

> “I would start with the least-privileged built-in, virtual, or managed identity that meets the requirements. I would avoid LocalSystem unless there is a documented need, and I would separate service identities when auditability or resource isolation matters.”

### What is Session 0 isolation?

Windows services run in **Session 0**, a non-interactive session. Since Windows Vista, interactive users run in separate sessions, such as Session 1 or later. This is called **Session 0 isolation**.

The purpose is security: a privileged service must not expose its windows, input queues, or UI to a less-privileged desktop user, and a desktop user must not be able to inject input into a privileged service's UI.

Consequences:

- A service should not display dialogs, message boxes, tray icons, or normal desktop windows.
- “Allow service to interact with desktop” is obsolete and should not be used for modern designs.
- A service that needs UI should use a separate **user-session agent**.
- The service and UI agent should communicate over authenticated **IPC**, such as named pipes, RPC, sockets, or another explicitly secured mechanism.

**Expected keyword:** **Session 0 isolation**. The design answer is **split the privileged service from the interactive UI and use secured IPC**.

### How does a Windows Service receive stop and shutdown notifications?

The service registers a **control handler** with SCM. SCM invokes that handler when it sends a control code, such as:

- `SERVICE_CONTROL_STOP` when an administrator requests a stop;
- `SERVICE_CONTROL_SHUTDOWN` during system shutdown, if the service accepts shutdown notification;
- `SERVICE_CONTROL_PRESHUTDOWN` when the service opts into an earlier, longer shutdown phase;
- pause, continue, parameter-change, or session-change controls when configured.

The handler should return quickly. It should signal the main worker to stop, and the main service logic should perform cleanup outside the control callback where possible.

A normal stop sequence is:

1. Receive `STOP`.
2. Report **STOP_PENDING** with a wait hint/checkpoints if cleanup takes time.
3. Stop accepting new work and signal worker threads.
4. Drain or cancel work according to the service's shutdown contract.
5. Close handles, sockets, and child processes.
6. Report **STOPPED** with an exit code.

The service should be **idempotent**: repeated stop requests or a stop arriving during startup should not corrupt state.

### How would you configure service recovery after a crash?

Configure **failure actions** in the service's recovery settings. Typical policies are:

- first failure: **restart the service**;
- second failure: restart again or run a diagnostic program;
- subsequent failures: restart, alert, or reboot only for a carefully justified system-critical component;
- reset the failure counter after a stable interval.

Using `sc.exe`, a conceptual example is:

```powershell
sc.exe failure MyAgent actions= restart/60000/restart/60000/""/0 reset= 86400
```

The exact command syntax is sensitive to spaces after the equals signs. PowerShell or an installer API can be easier to validate in deployment automation.

Recovery is not a substitute for diagnosing the crash. Also configure:

- **Event Log** entries and structured application logs;
- a meaningful service exit code;
- crash dumps where appropriate;
- bounded restart behavior to avoid a **crash loop**;
- health checks if the service can remain alive while functionally broken.

In an interview, mention that you would verify the policy in **Services → Properties → Recovery**, inspect the event logs, and test both an unexpected process termination and a clean administrative stop.

---

## Linux daemons

### What is a Linux daemon, and how does it differ from a Windows Service?

A **daemon** is a long-running background process on Linux. Historically, a daemon often detached from its terminal by forking, calling `setsid`, changing directory, closing file descriptors, and writing a PID file. Modern Linux applications commonly remain in the foreground and let **systemd** supervise them.

A Windows Service and a Linux daemon are conceptually similar but are not identical abstractions:

| Windows | Linux |
|---|---|
| Service registration is managed by **SCM** | Daemon lifecycle is commonly managed by **systemd** |
| Application reports service states through SCM APIs | Application readiness may be inferred or reported through **`sd_notify`** |
| Controls arrive as service control codes | Stop/reload commonly use **POSIX signals** |
| Configuration is commonly registered under the service Registry key | Configuration is commonly declared in a **unit file** |
| Service identity is configured through Windows accounts/tokens | Identity is configured through **UID/GID**, groups, and Linux security controls |

Not every daemon must be managed by systemd, and not every background process is a well-behaved daemon. The important distinction is the **supervision contract**.

### How does systemd manage the lifecycle of a service?

`systemd` is commonly the system's **PID 1** and service manager. It reads **unit files**, builds a dependency transaction, starts processes, tracks them in **cgroups**, captures logs through the journal, and applies restart and timeout policy.

Typical lifecycle:

1. `systemd` loads unit files from locations such as `/etc/systemd/system` and vendor directories.
2. A target or explicit `systemctl start` request pulls the service into a dependency transaction.
3. systemd resolves requirements and ordering constraints.
4. It creates the configured execution context: user, group, environment, working directory, namespaces, limits, and security restrictions.
5. It launches `ExecStart` and determines readiness according to `Type=`.
6. It monitors the service's cgroup and main process.
7. It sends stop/reload signals, enforces `TimeoutStopSec`, and can send `SIGKILL` if graceful termination fails.
8. It records exit status, logs, and restart attempts, then applies `Restart=` and start-rate limits.

Useful commands:

```bash
systemctl status myagent.service
systemctl start myagent.service
systemctl stop myagent.service
systemctl enable myagent.service       # enable at boot
systemctl disable myagent.service
journalctl -u myagent.service -b       # logs for the current boot
systemctl cat myagent.service
systemd-analyze verify myagent.service
```

**Keywords:** **unit file**, **PID 1**, **dependency transaction**, **cgroup supervision**, **journald**, **readiness**, **timeouts**, **restart policy**.

### System service versus user service

A **system service** is managed by the system instance of systemd, normally PID 1. It is appropriate for machine-wide infrastructure and can start before a user logs in. Unit files commonly live in `/etc/systemd/system` or `/usr/lib/systemd/system`.

A **user service** is managed by a per-user systemd instance and normally runs with that user's UID and permissions. It is appropriate for applications or agents belonging to one user. It is controlled with:

```bash
systemctl --user status myagent.service
systemctl --user enable --now myagent.service
```

By default, a user manager may exist only while the user has a session. **Lingering** (`loginctl enable-linger USER`) allows selected user services to run without an active login, subject to the machine's policy.

The key difference is not merely where the unit file is stored. It is the **manager instance**, **security identity**, **lifecycle scope**, and whether the service is expected to be available system-wide or only for one user.

### `Type=simple`, `Type=exec`, `Type=notify`, and `Type=forking`

`Type=` tells systemd how to interpret startup and readiness.

| Type | Startup/readiness contract | Best use |
|---|---|---|
| **`simple`** | The process started by `ExecStart` is considered started immediately; this is the default | A foreground process with no separate readiness handshake |
| **`exec`** | Similar to `simple`, but systemd waits for the `execve` step to succeed, making early executable/setup failures more accurately visible | Modern foreground services where launch errors should be reported precisely |
| **`notify`** | The service explicitly sends `READY=1` using `sd_notify` after initialization | Services with meaningful initialization and a real readiness point |
| **`forking`** | The original process forks and exits; the child becomes the daemon, often with `PIDFile=` | Legacy daemons that daemonize themselves |

For new software, prefer a **foreground process** with `Type=exec` or `Type=notify` rather than implementing traditional double-fork daemonization. Let systemd own process supervision and logging.

If using `Type=notify`, the service must send readiness only after dependencies, sockets, migrations, and other required initialization are complete. `WatchdogSec=` can add a watchdog contract when the process sends periodic `WATCHDOG=1` notifications.

### Configure restart, dependencies, and startup ordering

An example unit:

```ini
[Unit]
Description=Example Agent
Wants=network-online.target
After=network-online.target
Requires=example-backend.service
After=example-backend.service

[Service]
Type=notify
ExecStart=/opt/example/bin/example-agent
User=example
Group=example
Restart=on-failure
RestartSec=5s
TimeoutStartSec=30s
TimeoutStopSec=30s

[Install]
WantedBy=multi-user.target
```

Important distinctions:

- **`Wants=`** pulls in another unit but usually does not make this service fail if that unit fails.
- **`Requires=`** expresses a stronger requirement; failure or disappearance of the required unit can affect this service.
- **`After=`** establishes start/stop ordering but does not pull another unit in.
- **`Before=`** is the inverse ordering relationship.
- **`PartOf=`** propagates start/stop/restart operations without necessarily expressing a hard requirement.
- **`BindsTo=`** is stronger than `Requires=` when the bound unit disappears.
- **`Restart=on-failure`** restarts unexpected failures but not a clean administrative stop.
- **`Restart=always`** is broader and should be used deliberately.
- **`StartLimitIntervalSec=`** and **`StartLimitBurst=`** prevent an infinite rapid restart loop.

After editing a unit:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now example-agent.service
```

**Interview distinction:** `After=` controls **ordering**, while `Requires=`/`Wants=` control **relationship and activation**. `After=` alone does not start the other unit.

### Linux signals and handling `SIGTERM` and `SIGINT`

A Linux **signal** is an asynchronous notification delivered to a process or thread by the kernel or another authorized process. Common signals include:

- **`SIGTERM`**: polite termination request; the default signal systemd uses for stopping services;
- **`SIGINT`**: interrupt, commonly produced by `Ctrl+C` in a terminal;
- **`SIGHUP`**: commonly used by daemons to reload configuration;
- **`SIGQUIT`**: termination with a core dump by default;
- **`SIGKILL`**: forced termination; cannot be caught, blocked, or handled.

A well-behaved daemon should handle **`SIGTERM`** gracefully and often handle **`SIGINT`** the same way so foreground execution and service execution have consistent shutdown behavior. The handler should request shutdown and return quickly; it should not perform complex blocking work, allocate memory, or take locks if avoidable.

A robust design is:

1. Receive the signal.
2. Set a shutdown flag or write to a self-pipe/event mechanism.
3. Wake the main event loop.
4. Stop accepting new work.
5. Drain or cancel work with a deadline.
6. Close resources and exit with a meaningful status.

systemd normally sends `SIGTERM`, waits for `TimeoutStopSec`, and then sends **`SIGKILL`** if the service has not exited. Cleanup must therefore be **bounded** and safe to repeat.

### Investigating a service that fails immediately after startup

Use a layered approach:

1. Check the unit's state and recent exit status:

   ```bash
   systemctl status myagent.service
   systemctl show myagent.service -p Result -p ExecMainStatus -p ExecMainCode -p MainPID
   ```

2. Read the service logs, including the previous boot if relevant:

   ```bash
   journalctl -u myagent.service -b --no-pager
   journalctl -u myagent.service -b -1 --no-pager
   ```

3. Inspect the effective unit and drop-ins:

   ```bash
   systemctl cat myagent.service
   systemctl show myagent.service
   systemd-analyze verify myagent.service
   ```

4. Check the common causes:

   - wrong or relative **`ExecStart`** path;
   - missing executable permission or wrong architecture;
   - incorrect **`User=`**, group, filesystem, socket, or device permissions;
   - missing environment variables or a different `PATH` than an interactive shell;
   - wrong `WorkingDirectory=`;
   - unavailable dependency or incorrect startup ordering;
   - port already in use;
   - invalid configuration, migration failure, or missing secret;
   - SELinux/AppArmor denial;
   - incorrect `Type=` or readiness notification;
   - service exits normally but `Restart=` or `Type=` makes that outcome undesirable.

5. Reproduce under the same identity and environment:

   ```bash
   sudo -u example /opt/example/bin/example-agent
   ```

6. If necessary, inspect system calls with **`strace`**, inspect a core dump with **`coredumpctl`**, or temporarily increase application logging.

Do not begin by adding `Restart=always`. That can hide the original problem and create a restart loop. First identify the **exit code**, **journal error**, and **effective execution context**.

### Running as root versus using Linux capabilities

Running as **root** means UID 0 and grants very broad authority, including special treatment by many filesystem permission checks. It is simple but creates a large **blast radius** if the process or one of its dependencies is compromised.

**Linux capabilities** split parts of root's traditional privilege into smaller units. For example, `CAP_NET_BIND_SERVICE` allows binding to ports below 1024 without granting every root capability.

For a service, prefer:

- `User=` and `Group=` for a non-root identity;
- only the needed capabilities, such as `CAP_NET_BIND_SERVICE`;
- `CapabilityBoundingSet=` to remove unused capabilities;
- `AmbientCapabilities=` when a non-root executable must retain a capability across `execve`;
- `NoNewPrivileges=yes` where compatible;
- filesystem and device restrictions such as `ProtectSystem=`, `PrivateTmp=`, and `DevicePolicy=` where appropriate.

Capabilities are **least privilege**, not a complete sandbox. Combine them with correct file permissions, namespaces, seccomp/AppArmor/SELinux policy, secret handling, and a service-specific UID.

---

## Cross-platform OS integration

### Creating, inspecting, and managing users, groups, processes, and permissions

The platform concepts map roughly as follows:

| Concept | Windows | Linux |
|---|---|---|
| User identity | Local/domain account, SID, access token | UID, GID, supplementary groups |
| Group | Local/domain group, SID | Unix group |
| Process inspection | `Get-Process`, `tasklist`, Process Explorer | `ps`, `pgrep`, `top`, `systemctl status` |
| Process termination | `Stop-Process`, `taskkill` | `kill`, `pkill`, `systemctl stop` |
| Service inspection | `Get-Service`, `sc.exe query`, `services.msc` | `systemctl status`, `systemctl show` |
| Permissions | NTFS **ACLs**, SIDs, security descriptors | mode bits, POSIX ACLs, UID/GID, capabilities, MAC policy |
| Permission inspection | `icacls`, `Get-Acl`, `whoami /all` | `ls -l`, `getfacl`, `id`, `namei -l` |
| Privilege elevation | UAC, administrator token, service privileges | `sudo`, root, capabilities |
| Logs | Event Log, application logs | `journald`, syslog, application logs |

#### Windows examples

```powershell
# Users and groups
Get-LocalUser
New-LocalUser -Name svc-example
Get-LocalGroup
Add-LocalGroupMember -Group "Example Operators" -Member svc-example
whoami /all

# Processes and services
Get-Process
Get-Service -Name MyAgent
Start-Service MyAgent
Stop-Service MyAgent
sc.exe query MyAgent

# Permissions
Get-Acl C:\ProgramData\Example
icacls C:\ProgramData\Example
sc.exe sdshow MyAgent
```

Use **SIDs and ACLs**, not display names alone, when building durable permission logic. Check both the service's token and the ACL on every resource it accesses.

#### Linux examples

```bash
# Users and groups
getent passwd
getent group
id example
sudo useradd --system --no-create-home --shell /usr/sbin/nologin example
sudo usermod -aG example-readers example

# Processes and services
ps aux
pgrep -a example-agent
systemctl status example-agent.service
systemctl stop example-agent.service

# Permissions
ls -ld /var/lib/example
namei -l /var/lib/example/config.yaml
getfacl /var/lib/example
sudo -u example test -r /var/lib/example/config.yaml
```

The cross-platform engineering pattern is to keep application authorization policy in the application, while using the OS for **process isolation**, **identity**, **resource ACLs**, **secret protection**, and **lifecycle supervision**.

### If the agent must support macOS

The architecture should target **launchd**, not assume SCM or systemd semantics.

#### Service manager and lifecycle

macOS uses **launchd** and `launchctl`. A background component is usually one of:

- a **LaunchDaemon**: system-level, commonly installed under `/Library/LaunchDaemons`, often running as root or a dedicated user;
- a **LaunchAgent**: user-session component under `~/Library/LaunchAgents` or `/Library/LaunchAgents`, running in a user's login context.

The configuration is a **plist** containing fields such as `Label`, `ProgramArguments`, `RunAtLoad`, `KeepAlive`, `UserName`, `GroupName`, `WorkingDirectory`, `EnvironmentVariables`, and resource limits.

Example shape:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.agent</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Library/Example/bin/agent</string>
        <string>--foreground</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

Useful lifecycle commands vary by macOS version and domain, but the concepts are:

```bash
launchctl print system/com.example.agent
launchctl print gui/$(id -u)/com.example.agent
launchctl kickstart -k system/com.example.agent
launchctl bootout system /Library/LaunchDaemons/com.example.agent.plist
```

A macOS agent should normally run in the foreground and let launchd supervise it. Do not blindly port a systemd unit or implement a Linux-style double fork.

#### Architecture changes

For an application with both privileged work and UI, use separate components:

1. A user-facing **LaunchAgent** for menus, notifications, and user-session state.
2. A minimal **LaunchDaemon** or privileged helper only for operations that actually need elevated access.
3. Authenticated **XPC** or another platform-appropriate IPC mechanism between them.

For app-bundled login/background work, prefer Apple's supported **ServiceManagement / `SMAppService`** APIs where applicable rather than relying only on manually copying plist files.

#### Permissions model changes

macOS still has users, groups, UID/GID permissions, and POSIX ACLs, but production software must also account for:

- **code signing** and **notarization**;
- **System Integrity Protection (SIP)**;
- **TCC privacy permissions**, such as Full Disk Access, Contacts, Camera, or protected user data;
- **sandbox entitlements** when the app is sandboxed;
- **Keychain** access groups and secure secret storage;
- launchd domain ownership and user-session boundaries.

Root is not a universal solution on macOS. A root daemon may still be blocked by **TCC**, SIP, sandboxing, code-signing requirements, or an incorrect launchd domain. Request the narrowest entitlement and privilege needed, and keep privileged code small.

#### Interview-ready macOS answer

> “On macOS I would replace SCM/systemd integration with launchd and plist or ServiceManagement registration. I would distinguish LaunchAgents from LaunchDaemons, run in the foreground under launchd, use launchd restart and lifecycle semantics, and split UI from privileged work using XPC. The permission model still uses UID/GID and ACLs, but I would also design for code signing, notarization, TCC, SIP, sandbox entitlements, and Keychain access.”

---

## High-value interview traps and distinctions

### Process versus readiness

**A process existing does not prove the service is ready.** Mention explicit readiness: Windows **RUNNING**, systemd **`Type=notify` / `READY=1`**, or an application health check.

### Ordering versus dependency

**`After=` is ordering, not dependency.** The same idea appears in Windows dependency declarations: starting one thing before another is different from declaring that one requires the other to exist.

### Stop versus crash

A clean administrative stop should usually not trigger a restart. An unexpected exit should trigger **`Restart=on-failure`** or Windows **failure actions**, subject to rate limits and policy.

### Root versus least privilege

Do not answer “run it as root/LocalSystem because it is a service.” Explain **least privilege**, a dedicated identity, narrowly scoped resource permissions, and the security impact of compromise.

### UI versus background work

Do not put UI in a privileged service. Use a user-session process and authenticated **IPC**. On Windows this is driven by **Session 0 isolation**; on macOS it is commonly a LaunchAgent plus LaunchDaemon/helper; on Linux it may be a user service plus system service.

### A concise closing answer

> “Across operating systems, I treat the agent as a foreground application process with an explicit lifecycle contract. The platform manager owns startup, readiness, stop signals, supervision, logging, identity, dependencies, and recovery. I run it with least privilege, make shutdown bounded and idempotent, separate UI from privileged work, and use platform-native integration: SCM on Windows, systemd on Linux, and launchd/ServiceManagement on macOS.”

