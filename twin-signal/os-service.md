### Windows

Runtime / boot sequence

1. Bootloader loads windows kernel + boot-start drivers (`Start=0`) directly.
2. Kernel finishes init, I/O manager loads system-start drivers (`Start=1`).
3. `wininit.exe` starts `services.exe` (the [SCM](#service-control-manager-scm)) - User Mode.
4. SCM reads the [registry](#registry-location), builds a dependency graph, and starts:
   - Auto-start services in dependency order
   - Delayed-auto-start services shortly after (via a separate timer, off the critical boot path)
5. Manual-start services wait for something to request (or another service, an app, or a user via [`services.msc`/`sc start`](#manage-tool-scm)).
   - create service [process SIDs](#service-sids-scm) by [ImagePath](#file-permissions-scm)
   - excute main()/Program
6. Each service process calls `StartServiceCtrlDispatcher()` early in `main()`, registering a `ServiceMain` entry point per service name — this is how one .exe can host multiple services (like `svchost.exe`).
7. `ServiceMain` calls `RegisterServiceCtrlHandlerEx()` to receive [control codes](#lifecycle-scm) (stop, pause, shutdown, custom), then reports `SERVICE_RUNNING`.
8. SCM polls/expects periodic status updates during pending states (`dwWaitHint`, `dwCheckPoint`) — if a service doesn't respond in time, SCM considers it hung.

#### Registry location

All services (and drivers) are registered under: `HKLM\SYSTEM\CurrentControlSet\Services\<ServiceName>`
Key values under each service key:

```init
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
ErrorControl — what happens if the service fails to start (ignore/normal/severe/critical, affects boot behavior)
ObjectName — account the service runs as (e.g. `LocalSystem`, `NT AUTHORITY\NetworkService`, or a domain user)
DependOnService / DependOnGroup — dependency list
DisplayName, Description
FailureActions (binary blob) — recovery settings (restart/run command/reboot, reset period)
```

For svchost-hosted services, there's also a `Parameters` subkey with `ServiceDll` pointing to the DLL, since the actual code isn't in an executable at ImagePath — svchost.exe is the ImagePath, and it loads the DLL.

#### Service Control Manager (SCM)

Windows services are background processes that run independently of user login, managed by the Service Control Manager (SCM).

- SCM (Service Control Manager) — `services.exe`, runs at boot, starts/stops/monitors all services
- Service executable — implements `ServiceMain()` and a control handler to respond to start/stop/pause requests
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

File and folder permissions are primarily based on NTFS ACLs.
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
4. `systemd` reads [unit files](#unit-file), builds the [dependency](#dependencies-and-ordering-separate-concepts) graph, and starts units required by the boot targets.
5. Services that are installed but not enabled wait until something requests them, for example another unit, socket/timer activation, D-Bus activation, or a user/admin running [`systemctl start`](#manage-tool-systemd).
  - `systemd` creates/configures the service's cgroup
   - applies [`User=` / `Group=`](#service-account-systemd)filesystem restrictions
   - executes the command configured in `ExecStart=`, eventually running the application's `main()` / `Program.Main()`
6. Monitoring and supervid through the service's cgroup. [`Restart=`](#recovery-systemd). Sends `SIGTERM` to graceful shutdown. If it does not exit within `TimeoutStopSec=`, systemd can terminate it with `SIGKILL`.

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

#### Dependencies and ordering (separate concepts):

Enabled services are pulled into the boot transaction through relationships such as `WantedBy=multi-user.target` created by `systemctl enable`. Services are started according to dependency and ordering rules such as [`Wants=` / `Requires=`](#dependencies-systemd) and [`After=` / `Before=`](#ordering-systemd).

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

=== Logging: journald ===

journalctl -u myapp.service -f       # follow
journalctl -b                        # this boot
journalctl -p err --since "1 hour ago"
journalctl -xe                       # recent errors with explanations

=== Boot analysis ===
systemd-analyze                      # total boot time
systemd-analyze blame                # slowest units
systemd-analyze critical-chain

=== Timers (cron replacement) ===

# backup.timer
[Timer]
OnCalendar=daily
Persistent=true
[Install]
WantedBy=timers.target
```