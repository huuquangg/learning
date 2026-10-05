# Required Qualifications

- Proficiency in at least one relevant systems/native [language](#languages), such as Rust, Go, or a comparable language
- ~~(Preferred) Experience with JavaScript or Python (Typescript)~~
- ~~(Preferred) Experience with Rust or Go for systems-level or security-focused development~~

- Solid understanding of [networking](#networking-foundations) fundamentals: TCP/IP, DNS, routing, firewalls, VPN protocols, and/or ZTNA concepts.
- ~~(Preferred) Experience building agents/clients for RMM (remote monitoring and management), EDR/XDR, MDM, VPN, or ZTNA products.~~

- Hands-on experience [building](./interview.md#os-services), [shipping](./packaging.md), and maintaining [background services, daemons](os-service.md), system agents, or other privileged/low-footprint desktop software.
- Experience with OS-level programming concepts: services/daemons, permissions models, local user/group management, process management, and system APIs on Windows, macOS, and/or Linux.
- (Preferred) Experience with installers, silent deployment, code signing, auto-update systems, and large-scale software distribution
- (Preferred) Experience developing agents for multiple desktop/server operating systems, including [Windows](./windows-service.md#service-control-manager-scm), macOS, and [Linux](./linux-daemon.md)

---

- Experience designing and consuming REST APIs and integrating third-party systems, including identity providers and [SaaS platforms](./interview.md#3rd-party-integration).
- Working knowledge of identity and access management concepts: authentication (SSO, SAML, OIDC), authorization models (RBAC/ABAC), and directory services.
- ~~(Preferred) Direct experience integrating with identity providers (Entra ID/Azure AD, Okta, Ping, Google Workspace) or SaaS admin/security APIs (e.g., Google Workspace, Microsoft Graph, Slack, Salesforce)~~

---

- Familiarity with local data storage, secure [credential](./interview.md#credentials) storage, encryption at rest/in transit, and secrets management.
- Familiarity with cloud platforms such as Azure or AWS, and real-time communication protocols (WebSockets, gRPC, MQTT).

---

- ~~Ability to independently troubleshoot agent, network, identity-integration, and platform-specific issues~~
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

# 0–10 minutes: Introduction

## Tell us about yourself and your relevant experience? (100%)

> I’m Quang, a software engineer 2.5 yoe, distributed systems and security-related products.
> Previously, at OPSWAT, I worked on an IDS with 3 components: Enterprise, Site, and Sensor. I was involved in all 3 but most of my hands-on work was around building the Sensor component including its communication, lifecycle, package and deployment.
> Currently, I’m at AvePoint, working mainly on a cloud platform in the Microsoft Azure.
> I’m interested in this role because it’s aligned with my experience and my background, especially around system agents, networking, and OS-level behavior.

## Walk us through the product you worked on at OPSWAT. What did you personally own?

> IDS for OT industrial sector. It had 3 layers: Sensor, Site, and Enterprise.
> At the Sensor layer, I was involved in building the agent that captureing network traffic, identified industrial devices and protocols from vendors such as Siemens and Schneider and patch management. I was also involved in packaging installers.
> At the Site layer, I worked mainly on backend features for managing devices and connections, built visualization.
> Another area I worked on was policy management and enforcement. When a policy violation was detected, Site could generate alerts and integrate with external systems such as NAC solutions, firewalls, and Aruba ClearPass to take further actions. I also integrated with platforms such as ServiceNow and Cisco Meraki for data enrichment.
> At the Enterprise layer, I worked on parts of the centralized dashboard and configuration, so apply for multiple Sites.
> Overall, I worked across all three layers, but the Sensor was probably the most relevant to this position because it involved running long-lived agents on Windows and Linux, dealing with service lifecycle, permissions, recovery, communication with Site, and deployment.

---

# 15–55 minutes:

## OS Services

### Why build services?

> Originally, the Sensor application was deployed together with hardware appliances → over time the hardware became a significant expensive.
> So we needed a more flexible approach that we moved toward a software-agent model. Instead of requiring customers to deploy new hardware everywhere, the Sensor could run as small application directly on existing customer machines.
> Once it became a software agent, we had to care much more about service lifecycle, permissions, resource usage, recovery after crashes, installation and upgrades, and reliable communication with Site.
> That transition is actually where a lot of my work around Windows Services and Linux daemons came from.

### [Build](./os-service.md)

### [Privilege & permissions](./os-service.md)

### [Packaging & installation](./packaging.md#packaging--installation)

### [Update & deployment](./packaging.md#update--deployment)

### [Failure & recovery](./os-service.md#failure-and-recovery)

### [Maintaining & troubleshooting](./os-service.md#failure-and-recovery)

## Credentials

### Where is agent `Local data storage and Secure credential storage` data in local?

> (All data) Sqlite Datbase + (sensitive data) sqlitecypher encripted + DPAPI (windows encrypted cyperkey) + systemd-creds (Linux).

> Files permissions Windows (least privilege): NTFS ACL → dedicated service account; Linux: chown owner → chmod 600/750

### Do you know `Encryption in transit and Encryption in rest`?

> Encryption in transit protects data while it is travelling between systems, usually using TLS, such as HTTPS between an agent and backend.
> Encryption at rest protects stored data, for example a local database, cached sensitive information, or credentials stored on disk.

### How would you choose how components communicate?

> We used three mechanisms, chosen by the nature of the data:
> REST API for stateless request/response: configuration, queries, dashboard data. Simple, cacheable, easy to retry.
> Sockets for low-latency, high-frequency signals: realtime status. If one is lost, the next one replaces it, so occasional loss is acceptable.
> Message queue for critical data like alerts and asset events. Messages are persisted and acknowledged, so if the network drops between tiers, nothing is lost and the queue redelivers after reconnect. Because redelivery can cause duplicates, consumers dedupe with message IDs (idempotent processing).

```
Sensor                         Site/ Queue
  |                                 |
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
  |==== Encrypted TLS channel ======|
  |                                 |
  |---- HTTPS POST /handshake ----->| (HTTPS)
  |<--- Access token / config ------|
  |                                 |
  |---- HTTPS GET /ws ------------->| (WSS)
  |     Upgrade: websocket          |
  |<--- 101 Switching Protocols ----|
  |==== WebSocket over TLS (WSS) ===|
  |---- heartbeat ----------------->|
  |<--- command --------------------|
  |---- status/event -------------->|
  |                                 |
  |---- AMQP authenticate --------->| (AMQPS)
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
> The core concepts carry over: OOP, design pattern,...
> idioms, like goroutines in Go or ownership in Rust, I may take time to deep dive in.
> I also use AI to get familiar with language radpily so I dont find any problems here.

---

## Networking Foundations

| Topic          | Interview-ready understanding                                                                                                                                                                                                                                                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **TCP vs UDP** | Both are **transport-layer protocols**. **TCP** provides a reliable, ordered datagrams: it establishes a connection, tracks sequence numbers, acknowledges data, and retransmits lost data. **UDP** sends independent datagrams with no guarantee of delivery, tracks ordering, or retransmission, trading reliability for lower overhead and latency. |
| **DNS**        | DNS translates human-friendly domain names such as `google.com` into information computers can use, most commonly IP addresses such as `142.x.x.x`.                                                                                                                                                                                                    |
| **Routing**    | Network routing decides **which path packets take between networks**, based primarily on destination IP addresses and routing tables.                                                                                                                                                                                                                  |
| **Ports**      | An IP identifies a **machine/network interface**, while a port identifies a particular network service/process endpoint on that machine. For example `192.168.1.10:443`: IP → machine, port `443` → service listening there.                                                                                                                           |
| **Sockets**    | A socket is the **programming abstraction/API** applications use to communicate over the network. You can create a TCP socket or UDP socket. A network connection is commonly identified by protocol + source IP/port + destination IP/port.                                                                                                           |
| **HTTP**       | HTTP is an **application-layer request/response protocol**. HTTP/1.1 and HTTP/2 normally run over TCP. HTTP itself is stateless: each request contains the information needed to process it, although applications can maintain state using cookies, tokens, sessions, databases, etc. HTTP/3 is different: it runs over QUIC, which uses UDP.         |
| **WebSocket**  | WebSocket gives the client and server a **persistent, full-duplex connection**, allowing either side to send messages at any time. Important correction: traditional WebSocket normally runs over **TCP, not UDP**. It usually begins with an HTTP handshake and then upgrades the connection to WebSocket.                                            |
| **Firewall**   | A firewall enforces rules controlling network traffic. Rules can consider things like source/destination IP, ports, protocol, connection state, application, interface, etc., and decide whether traffic is allowed or blocked.                                                                                                                        |
| **VPN**        | It creates an encrypted tunnel between your device and another network/VPN gateway. It can route some or all network traffic through that tunnel, making your device logically connected to the remote/private network.                                                                                                                                |
| **ZTNA**       | Zero Trust Network Access provides access based on **identity, device posture, policy, and context**, rather than trusting someone merely because they are connected to the corporate network. Typically, it grants access to specific applications/resources instead of giving broad network access like a traditional VPN.                           |

### VPN vs ZTNA

> A traditional VPN establishes an encrypted tunnel and usually gives the device network-level access to a private network. ZTNA follows zero-trust principles: it continuously evaluates identity, device posture, and policy, and grants access to specific resources rather than implicitly trusting a device because it's inside the network.

---

## 3rd party integration

> I’ve integrated with several third-party systems.
> In my previous work, I integrated with platforms and SDKs such as ServiceNow, Cisco Meraki, Aruba ClearPass, Siemens, Schneider Electric, and Rockwell Automation. Depending on the integration, I worked with REST APIs or vendor SDKs, handled authentication, mapped external data into our internal models, processed errors and timeouts, and made sure the integration could recover when the external system was temporarily unavailable.

> "I haven't work with `not-use-tech` in production. The closest thing I've done is `<...>`. My understanding is that `not-use-tech` which `<Explain>`, so I wouldnt to claims that part."

---

# 55+ minutes

### Describe a deepfake project that you have worked on?

Remember: Forensic traces (face edge noise, light distribution) → human behavior (eye blink, gaze, head pose) → fuse both views → detect deepfake → alert.

### What are your availability, notice period, and expectations for remote collaboration?

My notice period is 30 days, starting from the day I accept the offer, so I could start right after that. I'm comfortable working remotely and I'm used to collaborating that way.
