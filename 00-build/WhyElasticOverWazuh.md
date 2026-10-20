# Why I Moved the Lab From Wazuh to Elastic Security
 
*2026-10-20*
 
Two days into building this lab, I replaced the SIEM I had just installed. Here's why, the alternatives I considered, and what I gave up.
 
## The goal the SIEM has to serve
 
This lab exists to practice **detection as code**: write a detection once, keep it in Git, test it, and deploy it automatically. I chose [Sigma](https://github.com/SigmaHQ/sigma) as the source format because it's vendor-neutral. The same rule can become KQL for Microsoft Sentinel, a query in a lab SIEM, or a query in any other platform.
 
So the SIEM's most important property isn't its dashboard. It's **how cleanly a Sigma rule becomes a running detection in it.**
 
## Where Wazuh fell short
 
I installed Wazuh all-in-one on a mini PC (Ubuntu Server, indexer heap tuned to 2 GB for an 8 GB host) and started planning the detection pipeline. Before enrolling a single agent, I hit the problem:
 
- **There is no official Sigma backend for Wazuh rules.** Sigma's converter (pySigma) produces *queries*. Wazuh alerts come from XML rules built around parent rule IDs (`if_sid`), regex field matches and numeric levels. That's a different model, and community converters cover only part of it.
- In practice, every detection would be translated by hand into Wazuh XML. That's fine for five rules, and a bottleneck for fifty.
- Wazuh's indexer is OpenSearch, so I *could* convert Sigma into search queries for hunting, but those don't fire alerts on their own.
Wazuh is a strong platform, especially for FIM, compliance checks and active response. It just wasn't built around the workflow and intent this lab is for.
 
## What I compared
 
Criteria: free to self-host at lab scale, real alerting, a maintained path from Sigma to running detections.
 
| Platform | Sigma path | Free for this lab? | Result |
| --- | --- | --- | --- |
| **Elastic Security** | Maintained pySigma backend (Lucene, EQL, ES\|QL), output importable as detection rules | Yes, Basic tier | **Chosen** |
| OpenSearch Security Analytics | Imports Sigma YAML natively | Yes, Apache 2.0 | Runner-up |
| Security Onion | Native Sigma via ElastAlert | Yes | Too heavy for my remaining RAM |
| Splunk Free | pySigma backend to SPL | 500 MB/day, alerting disabled | Ruled out |
| Graylog Open | Sigma appears limited to paid editions | Partly | Ruled out |
| LimaCharlie | Native Sigma | 2 endpoints, cloud-hosted | Too small |
| Grafana Loki | pySigma backend | Yes | Weak fit for Windows event logs |
| Wazuh | Manual translation to XML | Yes | Replaced |
 
## Why Elastic over OpenSearch
 
OpenSearch was the closest call. It imports Sigma directly and it's genuinely open source. Elastic won on four points:
 
1. **Detection as code is the skill I want to improve.** With Elastic, I run `sigma convert` with a field-mapping pipeline in CI and deploy the result. That conversion step, mapping Sigma's field names to the ECS schema, is real detection-engineering work. Native import would hide it.
2. **EQL sequence rules.** Elastic can express "this process, then this network connection, on the same host, within 30 seconds." Multi-event detections are where single-event rules stop being enough.
3. **Free EDR.** Elastic Defend gives the Windows VMs real endpoint telemetry and prevention at no cost.
## What I gave up
 
- **Open-source licensing.** Elastic's default distribution is under the Elastic License 2.0 / SSPL, with AGPLv3 also offered for Elasticsearch and Kibana. For a personal lab the free Basic tier covers everything I need, but it's not the same as Apache 2.0.
- **Wazuh's host features.** FIM with active response and built-in compliance checks. YARA moves to a scheduled scan that writes matches to the Windows event log, which Elastic Agent then ships.
- **Hardware placement.** Elastic runs as a 10 GB VM on the Proxmox host instead of on the mini PC. The mini PC's 2013-era dual-core CPU would struggle with indexing, and the hypervisor gives me snapshots. The G5 now runs 26 GB of VMs out of 32 GB.
## The safeguard
 
Because Sigma stays the source of truth, this decision is reversible. If Elastic's licensing or resource use becomes a problem, my rules move to OpenSearch with a new conversion pipeline, not a rewrite. That portability is the reason to use Sigma in the first place.

## Lessons
 
- **Pick the platform from the workflow, not the feature list.** I chose Wazuh for its feature list, then found it didn't fit the one workflow the lab exists for.
- **Check the integration path before building on a tool.** Twenty minutes researching "does Sigma convert to X" would have saved an evening of installing.
- **Changing course early is cheap.** No agents were enrolled and no detections were written, so switching cost one install.
