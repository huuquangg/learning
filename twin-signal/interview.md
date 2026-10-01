# Required Qualifications

[Languages](./interview.md#languages)

- Proficiency in at least one relevant systems/native language, such as Rust, Go, or a comparable language
- ~~(Preferred) Experience with JavaScript or Python (Typescript)~~
- ~~(Preferred) Experience with Rust or Go for systems-level or security-focused development~~

---

[Networking Foundations](./network.md#networking-foundations)

- Solid understanding of networking fundamentals: TCP/IP, DNS, routing, firewalls, VPN protocols, and/or ZTNA concepts.
- ~~(Preferred) Experience building agents/clients for RMM (remote monitoring and management), EDR/XDR, MDM, VPN, or ZTNA products.~~

---

[OS Services](./interview.md#os-services)

- Hands-on experience building, [shipping](./packaging.md), and maintaining background services, daemons, system agents, or other privileged/low-footprint desktop software.
- Experience with OS-level programming concepts: services/daemons, permissions models, local user/group management, process management, and system APIs on Windows, macOS, and/or Linux.
- (Preferred) Experience with installers, silent deployment, code signing, auto-update systems, and large-scale software distribution
- (Preferred) Experience developing agents for multiple desktop/server operating systems, including [Windows](./windows-service.md#service-control-manager-scm), macOS, and [Linux](./linux-daemon.md)

---

[3rd party intergration](./interview.md#3rd-party-integration)

- Experience designing and consuming REST APIs and integrating third-party systems, including identity providers and SaaS platforms.
- Working knowledge of identity and access management concepts: authentication (SSO, SAML, OIDC), authorization models (RBAC/ABAC), and directory services.
- ~~(Preferred) Direct experience integrating with identity providers (Entra ID/Azure AD, Okta, Ping, Google Workspace) or SaaS admin/security APIs (e.g., Google Workspace, Microsoft Graph, Slack, Salesforce)~~

---

[credentials](./interview.md#credentials)
[communication](./interview.md#communication)

- Familiarity with local data storage, secure credential storage, encryption at rest/in transit, and secrets management.
- Familiarity with cloud platforms such as Azure or AWS, and real-time communication protocols (WebSockets, gRPC, MQTT).

---

[scenarios](./interview.md#scenarios)

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


# 0–5 minutes: Introduction

## Tell us about yourself and your relevant experience? (100%)

> I’m Quang, a software engineer 2.5 yoe, distributed systems and security-related products. 
> Previously, at OPSWAT, I worked on an IDS with 3 components: Enterprise, Site, and Sensor. I was involved in both the Sensor agent and the Enterprise, Site backend, but most of my hands-on work was around building and maintaining the Sensor agent including its communication, service lifecycle, and deployment. 
> Currently, I’m at AvePoint, working mainly on a cloud platform in the Microsoft Azure ecosystem. 
> I’m interested in this role because it’s quite close to the kind of work I did, especially around system agents, networking, and OS-level behavior.

---

# 5–10 minutes: Your experience and project ownership.

## Walk us through the product you worked on at OPSWAT. What did you personally own?

> IDS for OT industrial sector. It had 3 layers: Sensor, Site, and Enterprise, data flowing upward from Sensor to Enterprise.
> At the Sensor layer (the data collection layer), I was involved in building and maintaining the agent lifecycle on both Windows and Linux. The agent captured network traffic and identified industrial devices and protocols from vendors such as Siemens and Schneider. I was also involved in packaging and deploying the agent across both platforms.
> At the Site layer, I worked mainly on backend features for managing devices and their profiles. I also built connection visualization so users could see relationships between assets as a graph.
> Another area I worked on was policy management and enforcement. When a policy violation was detected, Site could generate alerts and integrate with external systems such as NAC solutions, firewalls, and Aruba ClearPass to take further actions. I also integrated with platforms such as ServiceNow and Cisco Meraki for data enrichment and external workflows.
> At the Enterprise layer, I worked on parts of the centralized dashboard and configuration management, so administrators could manage multiple Sites and push configuration from one place.
> Overall, I worked across all three layers, but the Sensor side was probably the most relevant to this position because it involved running long-lived agents on Windows and Linux, dealing with service lifecycle, permissions, recovery, communication with Site, and deployment.

---

# 15–55 minutes:

## OS Services

### Why build services?

> Originally, the Sensor application was deployed together with hardware appliances > over time the hardware became a significant cost. Every new new customers required additional physical devices, logistics, maintenance, and replacement when hardware failed or became outdated.
> So we needed a more flexible approach that we moved toward a software-agent model. Instead of requiring customers to deploy new hardware everywhere, the Sensor could run as small application directly on existing customer machines.
> Once it became a software agent, we had to care much more about service lifecycle, permissions, resource usage, recovery after crashes, installation and upgrades, and reliable communication with Site.
> That transition is actually where a lot of my work around Windows Services and Linux daemons came from.

Build & lifecycle
### Tell me about a background service/agent you built?
Why did it need to be a service/daemon instead of a normal application? 
How does it start on Windows and Linux?
How does it stop gracefully?
What happens when the machine reboots?
How does the service communicate with the backend?
How do you handle long-running work without blocking the service?
Privilege & permissions
Which account did your service run under?
Why did it need that permission?
Why shouldn't everything run as LocalSystem/root?
How do file permissions work on Windows?
How do Linux user/group permissions work?

Packaging & installation
How did you package the Windows agent?
What did the installer actually do?
How was the service registered?
How did silent installation work?
How did Linux installation differ?
Update & deployment
How would you update an installed agent?
What if the update fails?
How would you deploy to thousands of machines?
How do you verify the update package?

Failure & recovery
Scenario A — process crash
Scenario B — backend unavailable
### What should happen when the backend is unavailable?

> I would use timeouts and retry temporary failures with increasing delays, up to a maximum delay. Some randomness in the delay helps devices avoid reconnecting together. If data must survive an outage, I would use a bounded local buffer and define what happens when it fills. After reconnecting, I would send pending work carefully. Authentication or invalid-request errors need investigation rather than repeated retries.

**Remember:** Timeout → backoff → bounded buffer → reconnect.

Scenario C — machine reboot
Scenario D — graceful shutdown
Maintaining & troubleshooting
How do you know the agent is healthy?
### How would you investigate high CPU or memory usage on a customer’s device?

> I'd start by finding out which process, version, and devices are affected, and when it began. On the device, I'd use `htop` on Linux or Task Manager on Windows to see what's using CPU or RAM and whether it keeps growing. I'd also add a simple health check to the agent that reports its own CPU and RAM every minute to our monitoring, with an alert if it stays above a threshold, and let systemd or the Windows service settings restart it if it crashes. Then I'd check the logs around when the problem began, and if needed, take memory dumps to see what's growing. To reproduce it, I'd run the same version and config in a test environment and leave it running for a few hours while watching CPU and memory. After fixing it, I'd add an alert so we catch it earlier next time.

How do you troubleshoot service restart loops?
Where are logs stored?
What if the service works manually but not as a service?
### Describe a difficult production bug. How did you find the root cause and verify the fix?

> We had a production issue where the connection between our components kept dropping, and it was hard to reproduce. Some devices would just stop sending data.
> Our socket was supposed to exist only once in the whole app, but the dependency injection setup was quietly creating a second copy without any error. One copy kept the live connection, and the other was the one some parts of the code were actually using. When the wrong one lost its connection, everything depending on it went silent.
> To find it, I looked at the logs around the drops and noticed the connection behaved as if it were two different objects. So I added logging in the constructor to see how many times it got created. It showed up twice, which confirmed the cause. Then I traced it back to how we registered it,it was registered in two different ways, so the system built one for each'.
> I fixed the registration so only one instance exists, then checked it by running the same scenario that used to fail, and the constructor now logged only once. Nothing dropped over.
> To prevent it from happening again, I added a test that checks the socket is created only once / added a startup check / documented how to register shared services.


### How would you keep an agent reliable over a long time?

> I would avoid busy loops, limit concurrent work and queue sizes, and release resources when they are no longer needed. I would add useful logs and monitor memory, CPU, and recent successful activity. Expected failures, such as a temporary network problem, should be handled. The service manager can restart a crashed process, but a process that is alive and stuck needs a separate health check.

### How would you make automatic updates safer?

> I would verify that the update comes from a trusted publisher and is intended for this platform and version before running it. I would preserve configuration, stop the service cleanly when required, install the update, and check that the service works afterward. I would also plan recovery if the update fails. I would not assume every installer automatically rolls everything back, especially if stored data has changed.

Remember: Verify → preserve → install → health check → recover.


## Credentials

`Local data storage and Secure credential storage`

### Where is agent storage data in local? 
- (All data) Sqlite Datbase + (sensitive data) sqlitecypher encripted + DPAPI (windows encrypted cyperkey) + systemd-creds (Linux).
- Files permissions Windows (least privilege): NTFS ACL → dedicated service account; Linux: chown owner → chmod 600/750

### Incase network/backend unavailable, how do you handle buffered data?
Remember: Backend down → RabbitMQ buffers. Network unreachable → agent buffers locally. Long time → buffered limit size or retention critical data.

### How to avoid corruption data when service crash/restart?
transaction/WAL → atomic write → persistent state → restart recovery → idempotent retry

`Encryption at rest / in transit`

### Do you know Encryption in transit and Encryption in rest?
- Encryption in transit protects data while it is travelling between systems, usually using TLS, such as HTTPS between an agent and backend.
- Encryption at rest protects stored data, for example a local database, cached sensitive information, or credentials stored on disk.  

### How would you choose how components communicate?

> We used three mechanisms, chosen by the nature of the data:
> REST API for stateless request/response: configuration, queries, dashboard data. Simple, cacheable, easy to retry.
> Sockets for low-latency, high-frequency signals: realtime status. If one is lost, the next one replaces it, so occasional loss is acceptable. 
> Message queue for critical data like alerts and asset events. Messages are persisted and acknowledged, so if the network drops between tiers, nothing is lost and the queue redelivers after reconnect. Because redelivery can cause duplicates, consumers dedupe with message IDs (idempotent processing).
```
Sensor                         Site/ Queue                     
  |                                 |
  |========== HTTPS / REST =========|
  |========== WebSocket ============|
  |========== Message Queue ========|
  |                                 |
  |---- Trust Company CA ---------->|
  |     (installed during setup)    |
  |                                 |
  |---- TCP connect --------------->|
  |---- TLS handshake ------------->|
  |<--- Server certificate ---------|
  |                                 |
  |---- Verify certificate ---------|
  |     - signed by trusted CA      |
  |     - hostname valid            |
  |     - not expired               |
  |                                 |
  |==== Encrypted TLS channel =====>|
  |                                 |
  |---- HTTPS POST /handshake ----->|
  |<--- Access token / config ------|
  |(--- HTTPS GET /inventory ------>|
  |<--- HTTPS response ------------)|
  |                                 |
  |                                 |
  |(--- HTTPS GET /ws ------------->|
  |     Upgrade: websocket          |
  |<--- 101 Switching Protocols ---)|
  |==== WebSocket over TLS (WSS) ===|
  |---- heartbeat ----------------->|
  |<--- command --------------------|
  |---- status/event -------------->|
  |                                 |
  |                                 |
  |---- AMQP authenticate --------->|
  |---- Publish event ------------->|
  |<--- Consume command/message ----|
  |---- ACK ------------------------|
```
`Secrets management`

> "There are Azure Key Vault, Vault but I haven't work with `not-use-tech` in production. As developer, we just consume it through API, this handle by Devops, so I wouldnt to claims that part.
---

## Languages

### Your strongest language is C#. How comfortable are you with Rust or Go?

> Claim the question, 
> but I don't see switching languages as a big obstacle. 
> The core concepts carry over: 
> idioms, like goroutines in Go or ownership in Rust, I may take time to deep dive in. 
> I also use AI to get familiar with language radpily so I dont find any problems here.

---


## 3rd party integration

> I have solid experience designing and consuming REST APIs, and I’ve integrated with several third-party systems.
> In my previous work, I integrated with platforms and SDKs such as ServiceNow, Cisco Meraki, Aruba ClearPass, Siemens, Schneider Electric, and Rockwell Automation. Depending on the integration, I worked with REST APIs or vendor SDKs, handled authentication, mapped external data into our internal models, processed errors and timeouts, and made sure the integration could recover when the external system was temporarily unavailable.

> "I haven't work with `not-use-tech` in production. The closest thing I've done is `<...>`. My understanding is that `not-use-tech` which `<Explain>`,  so I wouldnt to claims that part."
---

## 60+ minutes

### Describe a deepfake project that you have worked on?

Remember: Forensic traces (face edge noise, light distribution) → human behavior (eye blink, gaze, head pose) → fuse both views → detect deepfake → alert.

### What are your availability, notice period, and expectations for remote collaboration?

My notice period is 30 days, starting from the day I accept the offer, so I could start right after that. I'm comfortable working remotely and I'm used to collaborating that way.

