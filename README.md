# soc-home-lab — SOC home lab, built in public

I am putting together a SOC home lab and documenting the process in the open:
what I built, what I broke, and how I detected it.

Base: SOC fundamentals (SIEM, detection, response) on top of Wazuh. What makes
this lab different from the textbook setup is the focus on **modern
infrastructure** — containers, Kubernetes, and CI/CD pipelines.

## Roadmap

| # | Component | SOC skill it shows | Status |
|---|---|---|---|
| 1 | Wazuh base + SSH brute-force detection | SIEM, detection, ATT&CK | in progress |
| 2 | Endpoint hardening (sshd, ufw, fail2ban) | Linux / hardening | planned |
| 3 | Local privilege-escalation / persistence detection | Detection / ATT&CK | planned |
| 4 | Incident response (simulated case) | Incident response | planned |
| 5 | SOAR: Wazuh -> n8n -> Discord | Automation / SOAR | planned |
| 6 | Threat intel: IOCs + correlation | Threat intelligence | planned |
| 7 | Docker host monitoring | Containers (differentiator) | planned |
| 8 | Kubernetes monitoring (Kustomize + minikube) | K8s (differentiator) | planned |
| 9 | CI/CD: Jenkins pipeline + monitoring | CI-CD (differentiator) | planned |
| 10 | Azure endpoint reporting into the SIEM | Cloud | planned |
| 11 | Appendix: IDOR finding writeup (bug bounty) | - | planned |

## Architecture (local phase)

```
                  Host (laptop)
                  - hydra / curl / n8n
                          |
                          v  +---------------- lab-net (172.25.0.0/24) -----------------+
                          |   |                                                    |
                          |   |  172.25.0.10   172.25.0.20   172.25.0.30   172.25.0.40
                          |   |  victim        wazuh.indexer wazuh.manager  wazuh.dashboard
                          |   |  ubuntu+       (OpenSearch)  (manager+     (web UI)
                          |   |  sshd+                       agentd)
                          |   |  wazuh-agent                  ^
                          |   |  :22 (host 2222)              |
                          |   |                                |
                          |   +------ exposed: 1514, 55000, 5601 ---------------------+
                          +------------------------------------------------------>
```

Everything runs in Docker. See [`setup/README.md`](setup/README.md) for the
operator guide and [`docs/decisions/0002-migrate-vms-to-docker.md`](docs/decisions/0002-migrate-vms-to-docker.md)
for why this used to be two VirtualBox VMs.

## How the repo is organized

- `docs/decisions/` — ADRs: why I picked each thing and what I dropped.
- `docs/writeups/` — detections and incidents, mapped to MITRE ATT&CK.
- `bitacora/` — raw daily notes: what I tried, what broke, what I left for
  tomorrow.
- `setup/` — `docker-compose.yml`, the victim `Dockerfile`, and the agent /
  sshd configs to reproduce the stack.
- `incident-reports/` — incident response records (populated as the lab
  progresses).
- `docker/`, `kubernetes/`, `jenkins/`, `architecture/` — components of the
  differentiator, added as I get to them.

