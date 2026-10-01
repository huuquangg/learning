# Debian

```fresh-install
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
┌───────────────────────────────┐
│ Debian Package Installation   │
└───────────────┬───────────────┘
                │
                ▼
            agent.deb
                │
                │ apt install / dpkg -i
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
```

```upgrade
Existing installation
       │
       │ apt upgrade
       ▼
New agent.deb detected
       │
       ▼
prerm / package-manager hooks
       │
       ▼
Stop service if required
       │
       ▼
Preserve /etc configuration
       │
       ▼
Unpack new package
       │
       ├── replace /usr/bin/agent
       │
       ├── update supporting files
       │
       └── update systemd unit if changed
       ▼
postinst
       │
       ▼
systemctl daemon-reload
       │
       ▼
restart / start service
       │
       ▼
New version running
```

```
apt remove agent
      │
      ▼
prerm
      │
      ├── stop service
      └── disable service
      ▼
Remove package files
      │
      ▼
postrm
      │
      ▼
Package removed

/etc config may remain
```

# Windows

```
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
┌───────────┐
|   MSI     |
└─────┬─────┘
      |
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
```

## Packaging & installation

### How did you package the agent?

Remember: build the agent → collect/harvest artifacts → define metadata packaging → create MSI/EXE or DEB → install and configure the service → test fresh install, upgrade, and uninstall.

### What did the installer actually do?

Remember: install files → configure the agent → set permissions → register the service → enable automatic startup → start the service → handle upgrade and uninstall safely.

### How was the service registered?

- Windows: The MSI used WiX’s ServiceInstall to register the executable with the Service Control Manager, and ServiceControl to start or stop it during installation and removal.
- Linux: The .deb installed a systemd unit file under /lib/systemd/system/. The package scripts reloaded systemd and enabled the service so it could start on boot.

### How did silent installation work?

interactive install gets values from dialogs, while
silent install gets the same values from command-line msiexec with /qn or apt or dpkg to install the .deb.

## Update & deployment

### How would you update an installed agent?

On Windows: MSP patch or install a newer MSI.
On Linux, we’d publish a new .deb version. APT Repo handle

### What if the update fails?

signature verification fails: reject.
installation fails: rollback or a recovery to pervious.
After installation fails: restore and report the failure.

### How would you deploy to thousands of machines?

Ansible - Not best practises - Devops/IT handle this.

### How do you verify the update package?

signs it with a code-signing
-> MSI(Wixtoolset): ProductCode
-> Deb: GPG Key

### How would you investigate high CPU or memory usage on a customer’s device?

> Usually customer not allow us access into there enviroment → setting meeting to troubleshooting.
> Check htop or Task Manager on Windows → CPU or RAM and whether it keeps growing.
> Check systemctl or scm to see service health or crash loop.
> Check logs, memory dumps if it need.
> Reproduce
> Hot fix, release patch, workaround,...

### Describe a difficult production bug. How did you find the root cause and verify the fix?

> We had a production issue where the connection between our components kept dropping, and it was hard to reproduce. Some devices would just stop sending data.
> Our socket was supposed to exist only once in the whole app, but the dependency injection setup was quietly creating a second copy without any error. One copy kept the live connection, and the other was the one some parts of the code were actually using. When the wrong one lost its connection, everything depending on it went silent.
> To find it, I looked at the logs around the drops and noticed the connection behaved as if it were two different objects. So I added logging in the constructor to see how many times it got created. It showed up twice, which confirmed the cause. Then I traced it back to how we registered it,it was registered in two different ways, so the system built one for each'.
> I fixed the registration so only one instance exists, then checked it by running the same scenario that used to fail, and the constructor now logged only once. Nothing dropped over.
> To prevent it from happening again, I added a test that checks the socket is created only once / added a startup check / documented how to register shared services.
