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
**Agentless (Sonar/Agentless Gateway):**
For sources like cloud‑managed databases, the Agentless Gateway collects and enhances audit data.
It then forwards this data directly to the DSF Hub.

**-The DSF Hub ingests and stores this data centrally.**
**-Policies, configurations, and security rules are managed from the Hub.**
-Data is stored in the DSF Hub.
-Policies and monitoring configurations are managed through the USC (Unified Settings Console).
-A single dashboard provides unified visibility.
-Optionally, Data Risk Analytics (DRA) can be added — it applies ML‑based behavioral analytics on audit data to generate actionable incidents.
-On‑demand Kibana visualizations and audit search are also available.
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

**############################################################################################################**

**Phase 5 — Onboard Assets in DSF**
Step 5.1 — Create Azure cloud account in USC----> cloud account--->Azure cloud--->Enter details(subscription,sub_id, Select auth mechanism,and the Gateway server.).
Step 5.1 — Create Azure Event Hub Asset in USC----> Log Aggregator--->Assetname-->Enter detals(subcription name and id of event hub, storage assesskey , event hub assesskey, policy name,event hub access policy, storage container,event hub name , and namespace name and hostname, storage account name, and select parent asset.).
Step 5.1 — Create datasource Asset in USC----> Datasource--->Name---> Enterdetails(LogicalName, subcription name , id, db hostname, location, auth mechanism password:admin:admin, then select the parent assset.)

-In DSF, create an AZURE EVENTHUB asset. Required details:
eventhub\_name
eventhub\_namespace
azure\_storage\_account
azure\_storage\_container
format = Postgresql\_Flexible (DSF 4.15+)

-If using Managed Identity, set auth\_mechanism = azure\_ad in the Event Hub asset. For user-assigned identity, provide user\_identity\_client\_id.

Step 5.3 — Or Use Key-Based Authentication
\-If not using managed identity, provide:
eventhub\_access\_policy
eventhub\_access\_key
azure\_storage\_secret\_key

Step 5.4 — Create PostgreSQL Flexible Assets
In DSF, create an AZURE POSTGRESQL FLEXIBLE asset for each database server. Default port is 5432. You’ll need hostname, database\_name, username, and password for DB connection.
Step 5.5 — Map PostgreSQL Assets to Event Hub Asset
Map each PostgreSQL Flexible asset to the corresponding Event Hub asset ID (logs\_destination\_asset\_id). This relation is defined in the DSF asset specification.

Step 5.6 — Optional Azure Cloud Account Asset
If you plan to use discovery for auto-discovering assets, create an AZURE cloud account asset (with managed\_identity, client\_secret, or auth\_file).


**############################################################################################################**
**Phase 6 — Connect Gateway and Start Audit Collection**
Step 6.1 — Select Gateway in USC / Asset Dashboard
In DSF, go to the Unified Settings Console (USC) or the asset onboarding flow and select the Gateway. DSF onboarding documentation specifies that when saving a data source, you must assign a Gateway.

Step 6.2 — Save Event Hub / DB Assets
Save the Event Hub asset and PostgreSQL Flexible assets. DSF docs mention onboarding methods such as USC, Spreadsheet, or APIs.

Step 6.3 — Enable Audit Collection / Connect Gateway
For each relevant PostgreSQL Flexible asset, run the Enable Audit Collection or Connect Gateway action. In DSF onboarding flow, this step initiates audit log collection.
**############################################################################################################**
**Phase 7 — Validation (What to Check After Implementation)**
Step 7.1 — Verify Logs Are Reaching Event Hub
Check that diagnostic settings are active on each server and that events are being published to the Event Hub. Since Event Hub is the ingestion endpoint, event traffic should be visible there.

Step 7.2 — Check DSF Gateway Logs
On the Gateway host, review the following logs:
$JSONAR\_LOGDIR/eventhub/eventhub.log
$JSONAR\_LOGDIR/gateway/syslog/postgresql\_flexible\_eventhub.log

Step 7.3 — Verify Gateway Services Status
Check the status of Gateway services to ensure they are running properly and actively pulling audit logs.

Step 7.4 — Verify Source‑wise Logs in DSF
DSF doesn’t just ingest logs blindly. It tags each event with metadata like LogicalServerName or \_ResourceId.
That way, when you look at logs inside DSF, you can tell which PostgreSQL Flexible Server they came from.
Scenario: Imagine you have 15 branches of a bank, all sending daily transaction reports into one central inbox (Event Hub). DSF adds a label on each report saying “Branch #7” or “Branch #12.” When you review them, you know exactly which branch generated which report.

Step 7.5 — Validate Policy / Monitoring
The whole point of onboarding into DSF is not just collecting logs, but triggering audit/policy actions.
After ingestion, you should check if DSF Hub is raising alerts, enforcing policies, or showing audit analysis as expected.
Scenario: Continuing the bank analogy — it’s not enough that the reports arrive. You also want your compliance team to flag suspicious transactions. DSF is that compliance engine, so you verify that alerts are firing when rules are violated.

Special Notes (Key Reminders)
One Event Hub is enough for all 15 servers (same type).
→ Like one central inbox for all branches.
But 15 diagnostic settings are required (one per server).
→ Each branch must be configured to send its reports into the inbox.
One Blob Container is enough (checkpointing per Event Hub).
→ One shelf in the warehouse for sticky notes, even if 15 branches send reports.
Event Hub = temporary retention, Storage = checkpoint metadata.
→ Inbox holds reports for a short time; warehouse shelf holds sticky notes about progress.
Ideally one Gateway pulls from one Event Hub.
→ One clerk assigned to one inbox. If multiple clerks try to pull from the same inbox, they might duplicate or miss reports.
**###########################################################################**

```

🔹 Ultra‑Short Implementation Checklist (Roles Divided)
Azure Team
Create Event Hub Namespace
Create 1 Event Hub
Create Storage Account + 1 Blob Container
Assign Managed Identity to Gateway VM
Grant roles: Event Hubs Data Receiver + Storage Blob Data Contributor
Open ports 5671, 5672, 443

DBA Team
Enable pgaudit on servers
Run CREATE EXTENSION pgaudit;
Configure pgaudit.log + other log parameters
On each server, add Diagnostic Setting → PostgreSQLLogs → same Event Hub

DSF Team
Create AZURE EVENTHUB asset in DSF
Create AZURE POSTGRESQL FLEXIBLE assets
Set Event Hub asset format = Postgresql\_Flexible (DSF 4.15+)
Assign Gateway
Enable Audit Collection / Connect Gateway
