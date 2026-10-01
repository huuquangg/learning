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

All services (and drivers) are registered under: HKLM\SYSTEM\CurrentControlSet\Services\<ServiceName>
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

```

```
