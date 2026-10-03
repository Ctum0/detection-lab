# VM and asset inventory

## Virtual machines (Proxmox)

| VMID | Name | Role | OS | vCPU | RAM | Disk | Agent |
|---|---|---|---|---|---|---|---|
| 100 | linux-victim | Linux victim | Ubuntu 22.04.5 | 2 | 2 GB | 32 GB | Wazuh |
| 101 | windows-victim | Windows victim | Windows 10 Pro | 2 | 4 GB | 32 GB | Wazuh + Sysmon |
| 8000 | ubuntu-cloud | Cloud-init template | Ubuntu | 2 | 2 GB | 10 GB | none |

## Physical

| Asset | Role |
|---|---|
| Proxmox host (4 vCPU, 16 GB) | Hypervisor |
| Analyst laptop | Attacker and analyst workstation |

## VPS

| Service | Detail |
|---|---|
| Wazuh 4.14.8 | docker single-node (manager, indexer, dashboard) |
| n8n | SOAR workflow engine |
| Splunk | Secondary SIEM and hunting |
| Grafana | Metrics |

## Planned

| Asset | Purpose | Phase |
|---|---|---|
| dc-01 | Active Directory domain controller | Adversary Emulation, AD phase (planned) |
| win-member-01 | Domain member | Adversary Emulation, AD phase (planned) |
| sensor | Suricata network telemetry | Not yet scoped |
| honeypot | Cowrie / T-Pot | Threat Intelligence |
| TheHive / MISP / OpenCTI | Case management and threat-intel platform | SOAR / Threat Intelligence |
