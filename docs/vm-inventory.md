# CTUM — VM & Asset Inventory

## Virtual Machines (Proxmox)

| VMID | Name | Role | OS | vCPU | RAM | Disk | IP (LAN) | Tailscale | Agent |
|---|---|---|---|---|---|---|---|---|---|
| 100 | linux-victim | Linux victim | Ubuntu 22.04.5 | 2 | 2GB | 32GB | 192.168.1.112 | 100.109.150.66 | Wazuh |
| 101 | windows-victim | Windows victim | Windows 10 Pro | 2 | 4GB | 32GB | 192.168.1.113 | 100.116.117.32 | Wazuh + Sysmon |
| 8000 | ubuntu-cloud | cloud-init template | Ubuntu | 2 | 2GB | 10GB | - | - | none |

## Physical

| Asset | Role | Tailscale |
|---|---|---|
| Proxmox host (4c/16GB) | Hypervisor | 100.76.100.37 |
| Parrot laptop | Attacker / analyst workbench | on tailnet |

## VPS (vmi3486973)

| Service | Detail |
|---|---|
| Wazuh 4.14.8 | docker single-node (manager/indexer/dashboard), ports 1514/1515 on 0.0.0.0 |
| Splunk | secondary SIEM / hunting |
| Grafana | metrics |
| Tailscale | 100.81.241.62 |

## Planned (later phases)

| Asset | Purpose | Phase |
|---|---|---|
| dc-01 | Active Directory DC | Flagship 2 (AD) |
| win-member-01 | Domain member | Flagship 2 (AD) |
| sensor | Suricata / network telemetry | TBD |
| honeypot | Cowrie/T-Pot | TI feedback loop |
| TheHive / MISP / OpenCTI | Case mgmt + TIP | Flagship 4 / TI |
