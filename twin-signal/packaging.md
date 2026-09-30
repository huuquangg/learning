For Debian Linux, we first built the agent into a native executable and then packaged the artifacts as a .deb. The debian/control file defined package metadata and dependencies, while debian/rules used debhelper to drive the build and packaging lifecycle. We installed the executable under /usr/bin, configuration under /etc, and included a systemd unit for the background service. During installation, package scripts handled things like creating the service account, setting permissions and enabling or starting the service. For upgrades, we preserved configuration, replaced the application binaries, reloaded systemd when necessary and restarted the service. We tested the final .deb by installing and upgrading it on supported Debian environments.

## Debian
Source Code
   │
   │ build
   ▼
Native Agent Binary
   │
   │ collect artifacts
   ▼
Packaging Layout
├── agent executable
├── config files
├── systemd unit
└── Debian metadata
     │
     ▼
debian/
├── control        → package metadata + dependencies
├── rules          → debhelper build/package lifecycle
├── install        → where files should be installed
├── conffiles      → config preservation
├── postinst       → after install
├── prerm          → before remove/upgrade
└── postrm         → after remove
     │
     │ dpkg-buildpackage / debhelper
     ▼
   agent.deb
     │
     │ apt install / dpkg -i
     ▼
┌───────────────────────────────┐
│ Debian Package Installation   │
└───────────────┬───────────────┘
                │
                ▼
        Resolve dependencies
                │
                ▼
          Unpack files
                │
        ┌───────┼────────┐
        ▼       ▼        ▼
   /usr/bin   /etc    systemd unit
    agent     config   /lib/systemd/
                         system/
                │
                ▼
       Create service account
                │
                ▼
        Set owner / permissions
                │
                ▼
        systemctl daemon-reload
                │
                ▼
         systemctl enable
                │
                ▼
          systemctl start
                │
                ▼
           Agent Running


## Windows

Rust / Go / C# / C++
        │
        │ build / publish
        ▼
dist/
├── Agent.exe
├── xxx.dll
├── config.json
└── other dependencies
        │
        │
        ▼
WiX project
├── Product.wxs
├── Files.wxs
├── Service.wxs
├── UI/
│   ├── WelcomeDlg.wxs
│   ├── ConfigDlg.wxs
│   └── InstallDirDlg.wxs
├── Localization.wxl
└── images / license.rtf
        │
        ▼
       MSI

msiexec Agent.msi
       │
       ▼
┌──────────────────────────┐
│ InstallUISequence        │
│                          │
│ Check requirements       │
│ Resolve directories      │
│ Show dialogs             │
│ Collect properties       │
└────────────┬─────────────┘
             │
             │ user clicks Install
             ▼
┌──────────────────────────┐
│ InstallExecuteSequence   │
│                          │
│ Validate                 │
│ Begin transaction        │
│ Stop old service         │
│ Copy files               │
│ Write registry           │
│ Register service         │
│ Start service            │
│ Commit                   │
└──────────────────────────┘