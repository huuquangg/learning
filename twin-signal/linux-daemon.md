
### Linux

Runtime / boot sequence

1. Bootloader (for example GRUB) loads the Linux kernel.
2. Kernel loads CPU, memory, devices/drivers, filesystem.
3. starts `systemd` as PID 1.
4. `systemd` reads [unit files](#systemd-unit-files) from locations such as `/usr/lib/systemd/system/` and `/etc/systemd/system/`, resolves dependencies, and builds the boot transaction/dependency graph.

5. `systemd` starts units required by the boot targets, generally progressing through:

- `sysinit.target` — early system initialization
- `basic.target` — basic userspace services
- `multi-user.target` — normal non-GUI multi-user system
- `graphical.target` — desktop/GUI systems, when applicable

5. Enabled services are pulled into the boot transaction through relationships such as `WantedBy=multi-user.target` created by `systemctl enable`. Services are started according to dependency and ordering rules such as [`Wants=` / `Requires=`](#dependencies-systemd) and [`After=` / `Before=`](#ordering-systemd).
6. Services that are installed but not enabled wait until something requests them, for example another unit, socket/timer activation, D-Bus activation, or a user/admin running [`systemctl start`](#manage-tool-systemd).

- `systemd` creates/configures the service's cgroup
- applies [`User=` / `Group=`](#service-account-systemd), capabilities, namespaces, resource limits, environment, and filesystem restrictions
- executes the command configured in [`ExecStart=`](#exec-systemd), eventually running the application's `main()` / `Program.Main()`

7. Unlike a Windows Service, a normal systemd service does **not** need to call an API equivalent to `StartServiceCtrlDispatcher()`. For `Type=simple`/`Type=exec`, the process started by `ExecStart=` is normally treated as the service's main process and tracked by systemd through its cgroup.
8. The service lifecycle depends on [`Type=`](#service-type-systemd):

- `Type=simple` — systemd considers the service started after the process is launched
- `Type=exec` — considered started once `execve()` successfully executes the configured program
- `Type=notify` — application initializes and explicitly sends `READY=1` to systemd using `sd_notify()`
- `Type=forking` — used by traditional daemons that fork and let the parent process exit

9. During runtime, systemd tracks the service process and its child processes through the service's cgroup. If the main process exits unexpectedly, [`Restart=`](#recovery-systemd) controls whether and when the service is restarted.
10. When a stop or shutdown is requested, systemd normally sends `SIGTERM` to the service. The application handles the signal and performs graceful shutdown. If it does not exit within `TimeoutStopSec=`, systemd can terminate it with `SIGKILL`.

equivalents: daemons are background processes, and **systemd** is the init system/service manager on most modern distros (replacing SysV init and Upstart).
Daemon fundamentals

- A daemon is a long-running background process, typically with no controlling terminal, named with a trailing `d` (`sshd`, `crond`, `systemd-journald`).
- Classic (SysV-style) daemonization: fork, `setsid()` to detach from the terminal, fork again (so it can't reacquire a TTY), `chdir("/")`, reset `umask`, close/redirect stdin/stdout/stderr to `/dev/null`, write a PID file.
- Modern approach: don't daemonize yourself. Run in the foreground and let systemd supervise (`Type=simple`), logging to stdout/stderr (captured by journald).

Init history

- SysV init: shell scripts in `/etc/init.d/`, runlevels 0-6, symlinks in `/etc/rc*.d/`, sequential startup.
- Upstart: event-driven (Ubuntu, briefly).
- systemd: parallel startup, dependency-based, socket/bus/timer activation, cgroup tracking. PID 1.

systemd core concepts

- Units are the objects systemd manages. Types:
  - `.service` — a daemon/process
  - `.socket` — socket activation
  - `.timer` — cron-like scheduling
  - `.target` — grouping/sync points (like runlevels)
  - `.mount`, `.automount`, `.device`, `.path`, `.slice`, `.scope`, `.swap`
- Targets: `multi-user.target` (~runlevel 3), `graphical.target` (~5), `rescue.target`, `default.target` (symlink to the boot target).
- cgroups: each service runs in its own cgroup, so systemd can track all child processes and kill them reliably on stop.

Unit file locations (precedence high to low)

```
/etc/systemd/system/          # admin-created/overrides
/run/systemd/system/          # runtime
/usr/lib/systemd/system/      # package-installed (don't edit)
~/.config/systemd/user/       # per-user units
```

Use drop-ins (`systemctl edit foo.service` → `/etc/systemd/system/foo.service.d/override.conf`) rather than editing packaged files.

Example service unit

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
- Restart: `no`, `on-failure`, `always`, `on-abnormal`, etc.
- ExecStartPre / ExecStartPost / ExecStop / ExecReload
- TimeoutStartSec / TimeoutStopSec
- User / Group / DynamicUser

Dependencies and ordering (separate concepts):

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

Management commands

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

Logging: journald

```
journalctl -u myapp.service -f       # follow
journalctl -b                        # this boot
journalctl -p err --since "1 hour ago"
journalctl -xe                       # recent errors with explanations
```

Boot analysis

```
systemd-analyze                      # total boot time
systemd-analyze blame                # slowest units
systemd-analyze critical-chain
```

Timers (cron replacement)

```ini
# backup.timer
[Timer]
OnCalendar=daily
Persistent=true
[Install]
WantedBy=timers.target
```

Paired with `backup.service`.

Socket activation: systemd listens on a port/socket and starts the service on first connection, passing the file descriptor. Enables on-demand start and zero-downtime restarts.
