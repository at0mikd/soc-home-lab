# ADR 0002: Migrate the lab from VirtualBox VMs to Docker containers

- **Date:** 2026-07-31
- **Status:** accepted

## Context

The lab was running on VirtualBox: a `wazuh` VM (all-in-one OVA) and an
`ubuntu-agent` VM. Both worked, but the host laptop started freezing once I
opened a browser, a few terminals and both VMs at the same time. The OVA
allocates around 4-6 GB of RAM on its own, and the second VM plus the host OS
left no headroom.

The original "VM exercise" was never the goal; it was a means. The goal is to
have a SIEM that ingests real `sshd` logs, runs detections, and produces
verifiable alerts. If a lighter stack lets me run that on the same laptop
without freezes, the VMs are an accident of history, not a design choice.

## Decision

Replace both VirtualBox VMs with Docker containers. The lab will be run with a
single `docker compose up` from `setup/`.

Components:

- `wazuh.manager` — `wazuh/wazuh-manager:4.14.5`, ports 1514/55000 exposed to
  the host.
- `wazuh.indexer` — `wazuh/wazuh-indexer:4.14.5`, behind a profile so it can be
  scaled down when not in use.
- `wazuh.dashboard` — `wazuh/wazuh-dashboard:4.14.5`, depends on the indexer
  being healthy.
- `victim` — `ubuntu:22.04` with `init: true`, runs `sshd` and `wazuh-agent`,
  pinned to version 4.14.5-1.

Networking: a single user-defined bridge network `lab-net` (subnet
`172.25.0.0/24`). The victim gets a fixed IP `172.25.0.10`. The attacker is the
host machine, running hydra against the bridge IP.

Persistence: named volumes. The Wazuh services use the volume split from the
official `wazuh/wazuh-docker` compose (`wazuh_etc`, `wazuh_logs`,
`wazuh_queue`, `wazuh_indexer_data`, `wazuh_dashboard_config`, ...), plus
`victim_data` and `victim_logs` for the victim. `docker compose down` keeps
the data; `down -v` resets everything.

## Options considered

1. **Keep both VMs, give the host more RAM** — not realistic. The laptop is
   fixed hardware and the freeze happens before the second VM finishes booting.
2. **Containerize only Wazuh, keep the victim as a VM** — partially solves the
   RAM problem (saves ~3 GB), but leaves the heaviest single resource (the
   second VM) on the host. Half-measure.
3. **Containerize everything, but use a single all-in-one Wazuh image** —
   lighter than three containers, but worse fidelity to production and harder
   to debug when one piece misbehaves.
4. **Skip VMs entirely, simulate SSH failure logs with a script** — fastest,
   but removes the "real `sshd` on a real-ish endpoint" property of the lab.
   The detection stops being real.

I picked **Docker, three Wazuh containers, Ubuntu container for the victim**.
This is option (2)'s full sibling, not option (3) or (4).

## Consequences

- **Win:** RAM usage drops from ~5-6 GB to ~2.5-3 GB. Cold start goes from a
  full VM boot to `docker compose up`. The `ECONNREFUSED :55000` problem from
  the 2026-07-01 bitácora disappears: `restart: unless-stopped` replaces the
  manual `systemctl enable` that the OVA forgot to set.
- **Win:** the lab becomes a versioned `docker-compose.yml` plus a Dockerfile,
  which is more readable and forkable than a binary `.vbox`.
- **Trade-off:** a container is not a real endpoint. The `sshd` inside the
  Ubuntu container shares the host kernel. For the SSH brute-force detection
  this does not matter (we read `/var/log/auth.log` inside the container), but
  anything that relied on a separate kernel, a separate `auditd` or a separate
  `systemd` instance will need to be re-thought. K8s and Docker monitoring
  components will keep that limitation in mind.
- **Trade-off:** losing the "VM exercise". The first sessions of the lab were
  about fighting VirtualBox, OVA quirks and DHCP. That learning is gone.
  The bitácoras from those sessions stay as historical record, but no new
  work will happen on that path.
- **Debt:** the Wazuh container set is heavy on first pull (multi-GB). The
  first `docker compose up` will take longer than a VM import on a fast
  connection, and much longer on a slow one. Worth documenting.
- **Debt:** the K8s component of the roadmap (item #8) is **not** addressed
  by this ADR. K8s with Kustomize + minikube is still scoped for after the
  detection and SOAR components land (roadmap items #1 and #5).

## What stays from the old setup

- The agent-manager key exchange done on 2026-07-01 is no longer valid
  (different IPs, different containers) and will be redone when the new stack
  is first brought up.
- The two bitácoras from 2026-07-01 (`bitacora/2026-07-01-enrolar-agente-ubuntu.md`
  and `bitacora/2026-07-01-wazuh-api-econnrefused-55000.md`) stay as is. They
  are the honest record of the VM phase. A footer note pointing at this ADR
  is added to both.
- The ADR 0001 (Wazuh as the SIEM) is unchanged. This ADR only changes the
  packaging of the SIEM, not the choice.
