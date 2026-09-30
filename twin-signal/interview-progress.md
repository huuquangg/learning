

# 0–5 minutes: Introduction

## Tell us about yourself and your relevant experience? (100%)

> I’m Quang, a software engineer with about two and a half years of experience, mainly working on distributed systems and security-related products. Previously, at OPSWAT, I worked on an industrial intrusion detection system with three main components: Enterprise, Site, and Sensor. I was involved in both the Sensor agent and the Site backend, but most of my hands-on work was around building and maintaining the Sensor agent on Windows and Linux, including its communication, service lifecycle, and deployment. Currently, I’m at AvePoint, working mainly on a cloud platform in the Microsoft Azure ecosystem. I’m interested in this role because it’s quite close to the kind of work I did at OPSWAT, especially around system agents, networking, and OS-level behavior.

---

# 5–10 minutes: Your experience and project ownership.

## Walk us through the product you worked on at OPSWAT. What did you personally own?

> The product was an intrusion detection system for OT and industrial environments. It had three main layers: Sensor, Site, and Enterprise, with data flowing upward from Sensor to Site and then Enterprise.
> At the Sensor layer, which was the data collection layer, I was involved in building and maintaining the agent lifecycle on both Windows and Linux. The agent captured network traffic, extracted and normalized information, and identified industrial devices and protocols from vendors such as Siemens and Schneider. I was also involved in packaging and deploying the agent across both platforms.
> At the Site layer, I worked mainly on backend features for managing devices and their profiles. I also built connection visualization so users could see relationships between assets as a graph.
> Another area I worked on was policy management and enforcement. When a policy violation was detected, Site could generate alerts and integrate with external systems such as NAC solutions, firewalls, and Aruba ClearPass to take further actions. We also integrated with platforms such as ServiceNow and Cisco Meraki for data enrichment and external workflows.
> At the Enterprise layer, I worked on parts of the centralized dashboard and configuration management, so administrators could manage multiple Sites and push configuration from one place.
> Overall, I worked across all three layers, but the Sensor side was probably the most relevant to this position because it involved running long-lived agents on Windows and Linux, dealing with service lifecycle, permissions, recovery, communication with Site, and deployment.

---

# 15–55 minutes:

## OS Services

## Why build services?

> Originally, the Sensor application was deployed together with hardware appliances. But over time the hardware became a significant cost. Every new new customers required additional physical devices, logistics, maintenance, and replacement when hardware failed or became outdated.
> So we needed a more flexible approach that we moved toward a software-agent model. Instead of requiring customers to deploy new hardware everywhere, the Sensor could run as small application directly on existing customer machines.
> Once it became a software agent, we had to care much more about service lifecycle, permissions, resource usage, recovery after crashes, installation and upgrades, and reliable communication with Site.
> That transition is actually where a lot of my work around Windows Services and Linux daemons came from.

Build & lifecycle
Tell me about a background service/agent you built.
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
Scenario C — machine reboot
Scenario D — graceful shutdown
Maintaining & troubleshooting
How do you know the agent is healthy?
How do you troubleshoot high CPU?
How do you troubleshoot high memory?
How do you troubleshoot service restart loops?
Where are logs stored?
What if the service works manually but not as a service?

## Credentials

Mục này họ thường **không kỳ vọng bạn là security engineer chuyên cryptography**. Với role endpoint/agent, họ muốn biết bạn có tư duy đúng về việc **agent lưu dữ liệu local và giữ secret an toàn**.

`Local data storage`

- Agent của bạn lưu dữ liệu local ở đâu? Sqlite Datbase + cypher encripted + DPAPI (windows encrypted cyperkey) + systemd-creds (Linux).
- Nếu network/backend unavailable thì bạn xử lý buffered data thế nào?
- Làm sao tránh corruption khi service crash/restart?
- Files permissions ACLs for Windows and chmod chown for Linux (600, 750, 755)

`Secure credential storage`

- Bạn sẽ lưu API token/password/client secret của agent ở đâu? Sqlite Datbase + cypher encripted + DPAPI (windows encrypted cyperkey) + systemd-creds (Linux).
- Trên Windows bạn biết cơ chế nào để bảo vệ credential? Expected: Windows DPAPI, ACL.
- Linux thì sao? Expected: restrictive file permissions (chmod, chown), secret stores/keyrings nếu phù hợp.
- Nếu attacker copy được config file sang máy khác thì họ có sử dụng secret đó được không?

`Encryption at rest / in transit`

- Encryption in transit protects data while it is travelling between systems, usually using TLS, such as HTTPS between an agent and backend.
- Encryption at rest protects stored data, for example a local database, cached sensitive information, or credentials stored on disk.  
  They solve different problems, so normally sensitive systems need both.

### How would you choose how components communicate?

> We used three mechanisms, chosen by the nature of the data: REST API for stateless request/response: configuration push, queries, dashboard data. Simple, cacheable, easy to retry. Sockets for low-latency, high-frequency signals: heartbeats and status. If one is lost, the next one replaces it, so occasional loss is acceptable. Message queue for critical data like alerts and asset events. Messages are persisted and acknowledged, so if the network drops between tiers, nothing is lost and the queue redelivers after reconnect. Because redelivery can cause duplicates, consumers dedupe with message IDs (idempotent processing).

`Secrets management`

- Azure Key Vault, Vault, Dev just consume through API, this handle by Devops

## Languages

### Your strongest language is C#. How comfortable are you with Rust or Go?

> C# is my strongest language, but I don't see switching languages as a big obstacle. The core concepts carry over: types, concurrency, memory, error handling, and API design. What changes is the idioms, like goroutines in Go or ownership in Rust, I may take time to deep dive in. I also use AI to get familiar with language radpily so I dont find any problems here.

---


## 3rd party integration

> I have solid experience designing and consuming REST APIs, and I’ve integrated with several third-party systems.
> In my previous work, I integrated with platforms and SDKs such as ServiceNow, Cisco Meraki, Aruba ClearPass, Siemens, Schneider Electric, and Rockwell Automation. Depending on the integration, I worked with REST APIs or vendor SDKs, handled authentication, mapped external data into our internal models, processed errors and timeouts, and made sure the integration could recover when the external system was temporarily unavailable.
> I haven’t directly implemented an identity-provider integration such as Okta or Entra ID yet, so I wouldnt to claims that part. However, I’m familiar with the anothers third party integration so I’m confident I could pick up that part quickly.

---

## Scenarios

### How would you investigate high CPU or memory usage on a customer’s device?

> I'd start by finding out which process, version, and devices are affected, and when it began. On the device, I'd use `htop` on Linux or Task Manager on Windows to see what's using CPU or RAM and whether it keeps growing. I'd also add a simple health check to the agent that reports its own CPU and RAM every minute to our monitoring, with an alert if it stays above a threshold, and let systemd or the Windows service settings restart it if it crashes. Then I'd check the logs around when the problem began, and if needed, take memory dumps to see what's growing. To reproduce it, I'd run the same version and config in a test environment and leave it running for a few hours while watching CPU and memory. After fixing it, I'd add an alert so we catch it earlier next time.

### How would you keep an agent reliable over a long time?

> I would avoid busy loops, limit concurrent work and queue sizes, and release resources when they are no longer needed. I would add useful logs and monitor memory, CPU, and recent successful activity. Expected failures, such as a temporary network problem, should be handled. The service manager can restart a crashed process, but a process that is alive and stuck needs a separate health check.

### What should happen when the backend is unavailable?

> I would use timeouts and retry temporary failures with increasing delays, up to a maximum delay. Some randomness in the delay helps devices avoid reconnecting together. If data must survive an outage, I would use a bounded local buffer and define what happens when it fills. After reconnecting, I would send pending work carefully. Authentication or invalid-request errors need investigation rather than repeated retries.

**Remember:** Timeout → backoff → bounded buffer → reconnect.

### How would you make automatic updates safer?

> I would verify that the update comes from a trusted publisher and is intended for this platform and version before running it. I would preserve configuration, stop the service cleanly when required, install the update, and check that the service works afterward. I would also plan recovery if the update fails. I would not assume every installer automatically rolls everything back, especially if stored data has changed.

**Remember:** Verify → preserve → install → health check → recover.

### Describe a difficult production bug. How did you find the root cause and verify the fix?

> We had a production issue where the connection between our components kept dropping, and it was hard to reproduce. Some devices would just stop sending data.
> Our socket was supposed to exist only once in the whole app, but the dependency injection setup was quietly creating a second copy without any error. One copy kept the live connection, and the other was the one some parts of the code were actually using. When the wrong one lost its connection, everything depending on it went silent.
> To find it, I looked at the logs around the drops and noticed the connection behaved as if it were two different objects. So I added logging in the constructor to see how many times it got created. It showed up twice, which confirmed the cause. Then I traced it back to how we registered it,it was registered in two different ways, so the system built one for each'.
> I fixed the registration so only one instance exists, then checked it by running the same scenario that used to fail, and the constructor now logged only once. Nothing dropped over.
> To prevent it from happening again, I added a test that checks the socket is created only once / added a startup check / documented how to register shared services.

## 60+ minutes

### Describe a deepfake project that you have worked on?

"I worked on a research project about spotting deepfakes, mainly to protect face login on phones. The problem is that fake videos are getting very realistic, and tools that look for just one kind of clue often fail when the video is compressed or filmed with different cameras.
So we looked at two kinds of clues together. First, tiny traces that the fake-generation process leaves in the image, which you can't see by eye but show up when you analyze the image's patterns and noise. Second, how the person behaves: how they blink, where their eyes look, and how their head moves. Fake videos still struggle to get these natural human movements right.

### What are your availability, notice period, and expectations for remote collaboration?

My notice period is 30 days, starting from the day I accept the offer, so I could start right after that. I'm comfortable working remotely and I'm used to collaborating that way.

### Question with no idea?

Kubernetes (never used):

> "I haven't run Kubernetes in production. The closest thing I've done is packaging and managing service lifecycles with WiX and .deb installers. My understanding is that Kubernetes automates deployment, scaling, and restarts of containers. I'd start with a small local cluster like minikube to learn the core concepts, then deploy a simple service. Is that the kind of scenario you have in mind?"

Kafka (never used):

> "I haven't used Kafka directly. I've worked with RabbitMQ message queues for reliable delivery between tiers, so I understand acknowledgments and redelivery. My understanding is that Kafka is a distributed log built for high throughput and replay. I'd read up on partitions and consumer groups first, then prototype with a small topic."
