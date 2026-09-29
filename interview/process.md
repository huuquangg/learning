## 0–5 minutes: Introduction
### 1. Tell us about yourself and your relevant experience? (100%) 2-3
I’m Quang, *a software engineer with two and a half years of experience* building distributed systems.
Previously, *At OPSWAT*, I worked on an intrusion detection system which is make of three components: Enterprise, Site, and Sensor. On the Sensor components, agents running on Windows and Linux that captured network traffic and sent it to Site. At Site, manage and modeling the data for asset, connection, policy, and vulnerability. Enterprise provided centralized management across multiple Sites.
I’m now at *AvePoint*, working on backend services for backup and migration data for a partner platform in the Microsoft Azure cloud ecosystem.
I’m interested in this role (SWE Desktop/Native) because it *algined with my experience* with background agents, distributed communication agents, networking, and security platform.

“What interests you about Twin Signal and this Desktop/Native role?” **close to what I did**,**build further expertise in this area**
“Your recent work is mainly backend development. How does it prepare you for building endpoint agents?” **share several common aspects**, **valuable knowledge when bring it on**

---

## 5–15 minutes: Your experience and project ownership.

### “Walk us through the architecture of the product you worked on at OPSWAT. What did you personally own?”
"The product ít was an intrusion detection system for industrial and enterprise networks. It had three tiers: Sensor, Site, and Enterprise. Data flowed upward from Sensor to Enterprise.
- Sensor is the data collection layer, At this layer I developed agents that capture packets and recognize Siemens, Schneider devices assets and the communication on specific network segments, packaged the installers for both platforms, WiX for Windows and .deb for Linux and distribute and mornitoring the service lifecycle.
- Site is the processing and management layer for one location. It receives normalized data from Sensors and builds the core model. I worked on manage devices and proflies of them. I also built connection visualization, showing relationships between assets as a graph. I worked on policy management and enforcement: when a policy was violated, Site could trigger a alert and block through 3rd party NAC, firewalls, and Aruba ClearPass.
- Enterprise sits on top and provides centralized management across Sites: consolidated visibility, and configuration governance. I built parts of the dashboard for the cross-Site view. I worked on centralized configuration management, so settings could be pushed to multiple Sites from one place."

### “Tell us about a Windows service or background service you developed.”
I developed Sensor agents that ran as background services on Windows and Debian Linux. On Windows, the Service Control Manager started and supervised the service. On Debian, systemd did that job.
The agent captures network traffic, normalizes the data, and sends it to Site over an authenticated connection. On Windows, it runs under the service account configured in the Service Control Manager (SCM), such as LocalSystem if the deployment requires it. On Linux, it runs as the user specified in the systemd unit. Access to the agent's files is controlled through file ownership and permissions (chown/chmod on Linux, NTFS ACLs on Windows). Packet capture requires the appropriate OS permissions.
When the service receives a stop request, it closes its connections and releases its resources. If the connection to Site drops, the agent retries. If the process fails, SCM or systemd can restart it when recovery is configured, and the agent reconnects to Site.

### “What was your involvement in MSI installers and software releases?” 
- How did installation and upgrades work?
- What happened if an upgrade failed?
I worked on packaging and release scenarios for both Windows and Linux agents. For a fresh install, we used an MSI on Windows and a DEB on Debian. Each installed the agent, registered its service, and then we checked that the service started and connected.
For an update, we delivered an MSP patch or a newer MSI on Windows, depending on the release. On Debian, we delivered a newer DEB package. We applied the update, preserved the existing configuration, restarted the service, and checked its health.
If an update failed, we checked the installer and service logs. MSI or MSP can roll back installer-managed changes; for a failed DEB upgrade, we checked the package state and could explicitly install the previous working version. In both cases, we verified that the agent was running and communicating afterward.”

“Describe a difficult production bug. How did you find the root cause and verify the fix?”
- What evidence did you collect?
- How did you prevent it from recurring?
---
