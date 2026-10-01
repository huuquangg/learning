### Windows

Runtime / boot sequence

1. Bootloader loads windows kernel + boot-start drivers (`Start=0`) directly.
2. Kernel finishes init, I/O manager loads system-start drivers (`Start=1`).
3. `wininit.exe` starts `services.exe` (the [SCM](#service-control-manager-scm)) - User Mode.
4. SCM reads the [registry](#registry-location),
   - reads DependOnService builds a dependency graph,
   - reads Start to determinate which kind of start (Manual-start services wait for something to request (or another service, an app, or a user via [`services.msc`/`sc start`]))
   - reads [ImagePath](#file-permissions-scm) to service [process SIDs](#service-sids-scm)
5. Execute `main() / Program`
6. Calls `StartServiceCtrlDispatcher()` , registering a `ServiceMain` SetServiceStatus(`SERVICE_START_PENDING`).
7. SCM polls periodic status updates during pending states (`dwWaitHint`, `dwCheckPoint`) — if a service doesn't respond in time, SCM considers it hung.
8. `ServiceMain` calls `RegisterServiceCtrlHandlerEx()`
   - get [control codes](#lifecycle-scm) (stop, pause, shutdown, custom),
   - SetServiceStatus(`SERVICE_RUNNING`).

#### Registry location

All services (and drivers) are registered under: `HKLM\SYSTEM\CurrentControlSet\Services\<ServiceName>`
Key values under each service key:

```
ImagePath — path to the executable (or driver .sys file)
Start — startup type:
  - `0` = Boot (driver, loaded by kernel loader)
  - `1` = System (driver, loaded by I/O subsystem)
  - `2` = Automatic
  - `3` = Manual
  - `4` = Disabled
Type — what kind of service:
  - `1` = Kernel driver
  - `2` = File system driver
  - `0x10` = Own process (`SERVICE_WIN32_OWN_PROCESS`)
  - `0x20` = Shared process (`SERVICE_WIN32_SHARE_PROCESS`, i.e. svchost-hosted)
  - `0x110`/`0x120` = interactive variants (rare, legacy)
ErrorControl — if the service fails to start (ignore/normal/severe/critical, affects boot behavior)
ObjectName — account runs as (e.g. `LocalSystem`, `NT AUTHORITY\NetworkService`, or a domain user)
DependOnService / DependOnGroup — dependency list
DisplayName,
Description
FailureActions - recovery settings (restart/run command/reboot, reset period)
```

For svchost-hosted services, there's also a `Parameters` subkey with `ServiceDll` pointing to the DLL, since the actual code isn't in an executable at ImagePath — svchost.exe is the ImagePath, and it loads the DLL.

#### Service Control Manager (SCM)

Windows services are background processes that run independently of user login, managed by the Service Control Manager (SCM).

- SCM (Service Control Manager) — `services.exe`, runs at boot, starts/stops/monitors all services
- Service executable — implements `Main()` and a control handler to respond to start/stop/pause requests
- Startup types:
  - Automatic — starts at boot
  - Automatic (Delayed Start) — starts shortly after boot, reduces startup contention
  - Manual — starts on demand
  - Disabled — can't be started
- Service accounts — context a service runs under:
  - `LocalSystem` — highest privilege, full OS access
  - `LocalService` — minimal privileges, network access as anonymous
  - `NetworkService` — minimal privileges, network access as machine account
  - Custom user/domain account — for specific permission needs

#### Service SIDs (SCM)

- Service SIDs / isolation — each service can get its own SID so its resources can be locked down independently of others sharing a host process
- Session 0 isolation (since Vista) — services run in Session 0, separate from user sessions, so they can't interact with the desktop
  Service hosting:
- Standalone `.exe` per service, or
- Shared `svchost.exe` processes — many Windows services are DLLs grouped into shared host processes to reduce overhead

#### Manage Tool (SCM)

Management tools:

- `services.msc` — GUI
- `sc.exe` — command-line (create, config, query, start, stop)
- `PowerShell` — `Get-Service`, `Start-Service`, `Stop-Service`, `Set-Service`, `New-Service`
- Registry — service configs live under `HKLM\SYSTEM\CurrentControlSet\Services`

#### Lifecycle (SCM)

Lifecycle/state machine: `Stopped → Start Pending → Running → Stop Pending → Stopped` (plus Pause/Continue states if supported)

Dependencies: services can declare dependencies on other services or drivers; SCM starts them in the correct order.

Recovery options: each service can define actions on failure (restart service, run a program, reboot machine), configurable per failure count.

#### File permissions (SCM)

File and folder permissions are primarily based on `NTFS ACLs`.
SYSTEM → Full Control
Administrators → Full Control
SensorServiceUser → Modify
Users → Read
Common basic permissions:

- Read — read files, view folders.
- Write — create/write files.
- Read & Execute — read and run executables.
- Modify — read + write + delete.
- Full Control — virtually unrestricted access, including changing permissions and ownership.
  An important point is that folder permissions are typically inherited by child files and folders.

---

### Linux

Runtime / boot sequence

1. Bootloader (for example GRUB) loads the Linux kernel.
2. Kernel loads CPU, memory, devices/drivers, filesystem.
3. starts [`systemd`](#systemd) as PID 1.
4. `systemd` reads [unit files](#unit-file):
   - reads Dependencies builds the [dependency](#dependencies-and-ordering) graph.
   - Services that are installed
   - creates/configures the service's cgroup (Something requests them, for example another unit, socket/timer activation, D-Bus activation, or a user/admin running [`systemctl start`]).
   - reads [`User=` / `Group=`] applies filesystem restrictions
   - reads `ExecStart=` create PID
   - reads `Type=`
5. Execute `main() / Program` and becomes active (running)
6. Polling sd_notify(WATCHDOG=1) if WatchdogSec is set
7. Monitoring and supervid through the service's cgroup.
8. Sends `SIGTERM` to graceful shutdown. If it does not exit within `TimeoutStopSec=`, systemd can terminate it with `SIGKILL`.

#### systemd

- Units are the objects systemd manages. Types:
  - `.service` — a daemon/process
  - `.socket` — socket activation
  - `.timer` — cron-like scheduling
  - `.target` — grouping/sync points (like runlevels)
  - `.mount`, `.automount`, `.device`, `.path`, `.slice`, `.scope`, `.swap`
- Targets: `multi-user.target` (~runlevel 3), `graphical.target` (~5), `rescue.target`, `default.target` (symlink to the boot target).
- cgroups: each service runs in its own cgroup, so systemd can track all child processes and kill them reliably on stop.

#### unit file

Unit file locations (precedence high to low)

```
/etc/systemd/system/          # admin-created/overrides
/run/systemd/system/          # runtime
/usr/lib/systemd/system/      # package-installed (don't edit)
~/.config/systemd/user/       # per-user units
```

Use drop-ins (`systemctl edit foo.service` → `/etc/systemd/system/foo.service.d/override.conf`) rather than editing packaged files.

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

Key `[Service]` options

- Type: `simple` (default), `exec`, `forking` (classic daemon, needs `PIDFile=`), `oneshot`, `notify` (service signals readiness via `sd_notify`), `dbus`, `idle`
  - `Type=simple` — systemd considers the service started after the process is launched
  - `Type=exec` — considered started once `execve()` successfully executes the configured program
  - `Type=notify` — application initializes and explicitly sends `READY=1` to systemd using `sd_notify()`
  - `Type=forking` — used by traditional daemons that fork and let the parent process exit
- Restart: `no`, `on-failure`, `always`, `on-abnormal`, etc.
- ExecStartPre / ExecStartPost / ExecStop / ExecReload
- TimeoutStartSec / TimeoutStopSec
- User / Group / DynamicUser

#### Dependencies and ordering:

Enabled services are pulled into the boot transaction through relationships such as `WantedBy=multi-user.target` created by `systemctl enable`.
Services are started according to dependency and ordering rules such as [`Wants=` / `Requires=`] and [`After=` / `Before=`].

- `Requires=`, `Wants=`, `BindsTo=`, `Conflicts=` — _what_ gets pulled in
- `After=`, `Before=` — _order_ only

Hardening options (a big systemd strength)

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

#### Management commands

systemctl <action>
journalctl <log>
systemd-analyze <analyze>

### Build

#### Why did it need to be a service/daemon instead of a normal application?

A standard application: a user actively opens to perform a task and then closes. Examples include web browsers, photo editing software.
A background service is a program that runs in the background, usually without a user interface. It can start automatically with the system, continue running after the user logs out, and handle tasks either continuously or on a schedule. Examples include Windows Services or Linux systemd daemons.

#### How does it start on Windows and Linux? (manual)

#### What happens when the machine reboots? (auto)

[Windows](#windows)
[Linux](#linux)

#### How does the service communicate with the backend?

[Credential](./interview.md#credentials)

#### How do you handle long-running work without blocking the service?

I allow independent work to run concurrently, with a limit on the number of active workers so the service doesn’t accept more work than it can handle. When capacity is reached, the business requirements determine whether new work is queued or rejected with a temporary busy response. I use timeouts and cancellation when the operation is no longer needed or when the caller disconnects. I only cancel existing work to make room for a newer request if the business rules explicitly allow the newer request to replace it.

### Privilege & permissions

#### Which account did your service run under?

#### Why did it need that permission?

#### Why shouldn't everything run as LocalSystem/root?

A practical downside of using accounts with limited privileges is the need to precisely assign and maintain the necessary permissions:

- Insufficient access rights: The service might fail to read configuration files, write logs or data, or access required registry keys, devices, or directories; specific Access Control Lists (ACLs) must be configured.
- Varying network access: LocalService typically accesses the network as an anonymous user, whereas NetworkService and LocalSystem use the machine's identity to access network resources. Custom local users often lack the appropriate domain identity to access network resources. (Microsoft Learn)
- Increased complexity: Installers or administrators must create and configure the account, as well as set up logon rights and file/directory permissions; this process requires automation when deploying across multiple machines.

#### How do file permissions work on Windows?

NTFS + ACLs

#### How do Linux user/group permissions work?

chmod + chown

### Failure and Recovery

Scenario A — process crash

#### Tell me about how your service recovers after an unexpected process crash.

Windows SCM or systemd can restart process flowing recovery policy configuration. Agent back to stable state then continue it jobs.

Scenario B — backend unavailable

#### What does the service do when the backend becomes unavailable?

Remember: Backend down → RabbitMQ buffers. Network unreachable → agent buffers locally. Long time → buffered limit size or retention critical data.

Scenario C — machine reboot

#### What happens to your service when the machine reboots?

configured start automatically with the system

Scenario D — graceful shutdown

#### How does your service handle a graceful shutdown?

Take the requests, service stop take new jobs, inform stop worker, recent take execute for a while or take checkpoint, close connection và file. if time out, keep status to restart continuely.

- Windows: SCM gửi lệnh stop; service báo trạng thái STOP_PENDING trong lúc dọn dẹp, rồi báo STOPPED.
- systemd: thường gửi SIGTERM; service dọn dẹp trong giới hạn TimeoutStopSec, sau đó systemd có thể buộc dừng nếu process không thoát.

Data Consistency
