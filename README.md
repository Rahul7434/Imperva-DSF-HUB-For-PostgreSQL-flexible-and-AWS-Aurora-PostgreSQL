# Imperva-DSF-HUB-For-PostgreSQL-flexible-and-AWS-Aurora-PostgreSQL

**What is DSF Hub?**
```
In short: DSF Hub is the central brain + data warehouse + single dashboard of Imperva Data Security Fabric (DSF). Audit data coming from agents or agentless gateways is stored here, and policies/settings can be managed from one place.
Imperva DSF provides unified visibility, control, automation, and insights for data sources across multi‑cloud, hybrid, and on‑prem environments — and at the center of it all is the DSF Hub.

**How does it work? (High‑level flow):
-Data sources across different environments (cloud, hybrid, on‑prem) generate audit logs.**
-Agents or agentless gateways collect this audit data.**

**Agent‑based (DAM):
Imperva Agents are installed on the database servers.
These agents send logs/events to the Agent Gateway (formerly called DAM Gateway).
The gateway parses and normalizes the data, then ingests it into the DSF Hub.
&#x09;**Agentless (Sonar/Agentless Gateway):**
&#x09;For sources like cloud‑managed databases, the Agentless Gateway collects and enhances audit data.
&#x09;It then forwards this data directly to the DSF Hub.

&#x20; **-The DSF Hub ingests and stores this data centrally.**
&#x20; **-Policies, configurations, and security rules are managed from the Hub.**
&#x09;-Data is stored in the DSF Hub.
&#x09;-Policies and monitoring configurations are managed through the USC (Unified Settings Console).
&#x09;-A single dashboard provides unified visibility.
&#x09;-Optionally, Data Risk Analytics (DRA) can be added — it applies ML‑based behavioral analytics on audit data to generate actionable incidents.
&#x09;-On‑demand Kibana visualizations and audit search are also available.
So in essence, DSF Hub centralizes all audit data — whether collected via agents or agentless gateways — and provides unified visibility, classification, and control over sensitive information across your enterprise.
```

**Typical Deployment Structure (Simplified)**
```
**Core Building Blocks:**
DSF Hub (VM/Cloud Instance): Centralized dashboard, policy management, data warehouse/ingestion.
Agentless Gateway(ies): For cloud databases, SaaS, or log‑based sources; collects audit data and sends it to the Hub.
DAM Stack (Agent Gateway + Agents + MX): For on‑prem or VM‑hosted databases; agents collect logs → Agent Gateway processes them → Hub ingests; MX (Management Server) handles management/policies.
DRA (Optional): Advanced analytics layer on Hub audit data (Admin + Analytics servers).

**Cloud/Infrastructure Deployment (IaC/Automation):
Use Imperva’s DSF Kit (Terraform repo) to auto‑deploy DSF Hub, Agentless Gateway, DAM (MX/Agent Gateway), and DRA — supported on AWS and Azure.
Terraform Registry – DSFHub provider: Connects to the DSF Hub API (typically https://<host>:8443) to manage assets/configurations directly from code.
Agent Gateway on AWS (module): Quickly spin up a DAM Agent Gateway as an EC2 instance.
Agentless Onboarding modules: Ready‑made modules for auto‑onboarding cloud data sources like RDS or Snowflake.

**Quick Guide: “When to Use What”**
Only cloud‑managed DBs (RDS, Azure SQL, Snowflake, etc.): Use Agentless Gateway + DSF Hub, add DRA if needed. Terraform modules simplify onboarding.
On‑prem/VM‑hosted DBs, high traffic/tight control: Install agents on DBs → Agent Gateway + MX → DSF Hub at the center → DRA for analytics.
Hybrid mix (cloud + on‑prem): Use both gateway types with a single Hub — unified visibility/policies from one dashboard.
Daily Usefulness of DSF Hub
Centralized alerts, incidents, reports → simplifies life for security, audit, and compliance teams.
Fast search/investigation → query retained audit data in seconds, with Kibana visualizations.
SIEM/SOAR integration → plug DSF Hub into your existing security stack.

**TL;DR**
DSF Hub = Central control + data warehouse + dashboard.
Two collection paths: Agent (Agent Gateway/MX) and Agentless (Agentless Gateway). Everything ends up in the Hub.
Analytics/Use‑cases: DRA for behavioral analytics; Kibana/Reports; SIEM integration.
Deployment: AWS/Azure auto‑deploy via Terraform (DSF Kit, Providers \& Modules).
```

\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\

                                                        **DSF Hub for Azure PostgreSQL flexible servers (Agentless)**

\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\

```
Phase 0 — Points to Confirm First**
The format = Postgresql\_Flexible for Event Hub assets in DSF is supported starting from DSF version 4.15+. So confirming the version first is important.

-In DSF, the database asset type is AZURE POSTGRESQL FLEXIBLE. The onboarding guide and asset specification are different for Flexible Server.
-According to DSF documentation, a single Event Hub is supported for multiple instances of the same server type.

**############################################################################################################**

**Phase 1 — Create Azure Resources**
**Step 1.1 — Create Event Hub Namespace**
-Create an Event Hubs Namespace in Azure. A namespace is a management container that holds one or more Event Hubs.
-Inside that namespace, create one Event Hub that will receive logs from PostgreSQL Flexible servers. An Event Hub is an append-only event log that temporarily retains events.
-Configure Event Hub retention properly because Event Hub is not permanent storage. Once retention expires, events are purged. If you need a longer replay window, set retention accordingly.
-Create a Storage Account. Azure Storage provides a namespace for blobs/files/tables/queues. For DSF, a blob container is required.
-Inside the Storage Account, create one Blob Container. DSF documentation states that one container per Event Hub is required for import/checkpointing. Since you have one Event Hub, one container is generally sufficient.

**############################################################################################################**

**Phase 2 — Gateway Identity, Roles, Networking:
-Attach either a system-assigned or user-assigned managed identity to the DSF Agentless Gateway VM. DSF supports Azure AD / managed identity-based authentication.
-Grant the Gateway’s identity the Azure Event Hubs Data Receiver role so the Gateway can receive/pull logs from Event Hub.
-Grant the same identity the Storage Blob Data Contributor role so the Gateway can use the storage container for checkpoint/metadata.
-If the Event Hub Namespace uses selected networks, private endpoints, or firewall restrictions, allow the DSF Gateway subnet/vNet. Networking controls apply at the namespace level.
-Similarly, allow the DSF Gateway subnet/vNet on the Storage Account, since the Gateway needs HTTPS (port 443) access for storage metadata/checkpointing.

**Step 2.6 — Open Firewall Ports**
**Allow outbound access from the Gateway:**
5671 / 5672 → For Event Hubs AMQP / AMQPS
443 → For Storage Account HTTPS calls
In the firewall, allow the Event Hub Namespace hostname/FQDN (e.g., namespace.servicebus.windows.net), since Event Hub IPs can be dynamic.


The DSF Gateway VM connects to the Event Hub over AMQP/AMQPS protocols.
That means outbound traffic from the Gateway must be allowed on ports 5671 (secure TLS) and optionally 5672 (non‑TLS).
You’ve granted the Event Hubs Data Receiver role to the Gateway’s managed identity, so it has permission to pull logs from the Event Hub.
The Gateway also needs to talk to the Storage Account over HTTPS (port 443).
This is not for storing the logs themselves — Event Hub already holds them temporarily.
Instead, the Storage Account’s Blob Container is used for checkpointing metadata.
The Gateway writes small files there to mark “I’ve read logs up to this point.

**############################################################################################################**
**Phase 3 — Enable Audit on Each PostgreSQL Flexible Server**
-Enable/allowlist/load the pgaudit extension on Azure PostgreSQL Flexible Server for audit logging. According to Azure docs, PostgreSQL audit logging is based on pgaudit.
-Connect to each relevant database and run the command to create the pgaudit extension. Azure documentation specifies that creating the extension is required to use pgaudit.
-Set pgaudit.log according to your use case — for example, WRITE, or for broader visibility use READ, WRITE, DDL, ROLE classes. Azure docs note that required classes must be specified individually.
-Configure parameters such as log\_connections, log\_disconnections, log\_duration, log\_statement as needed. Azure docs explain that these standard PostgreSQL logging parameters are useful for security, troubleshooting, and audit visibility, but high-volume logging can impact performance.

Step 3.5 — Verify Audit Logging
Check whether DB logs are being generated. According to Azure docs, Flexible Server logs are available under the PostgreSQLLogs category.

############################################################################################################**
Phase 4 — Create Diagnostic Setting for Each Server**
Step 4.1 — Open Diagnostic Settings on Each PostgreSQL Flexible Server
In the Azure Portal, go to each Flexible Server resource → Diagnostic Settings. Azure Monitor docs state that diagnostic settings are independent for each resource.
-Add a diagnostic setting on each server. A resource can have multiple diagnostic settings, but for DSF flow, the Event Hub destination is required.
-Choose the PostgreSQLLogs category. This is the main category for Azure PostgreSQL Flexible logs.
-In each server’s diagnostic setting, select the same Event Hub as the destination. This ensures logs from all servers are streamed into one Event Hub. Azure docs confirm that logs can be streamed to Event Hub destinations.
-Save the diagnostic setting on each servers. Important note: multiple servers =  diagnostic settings, even if the Event Hub is common, because diagnostic settings are resource-specific.
```
