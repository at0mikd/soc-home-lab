# `setup/` — running the lab from Docker

This directory brings up the entire SOC home lab with a single
`docker compose up`. It replaces the VirtualBox-based setup that the previous
sessions of the repo documented. The reasoning lives in
[`docs/decisions/0002-migrate-vms-to-docker.md`](../docs/decisions/0002-migrate-vms-to-docker.md);
read that first if anything below feels arbitrary.

## What you get

- `wazuh.indexer` (172.25.0.20) — OpenSearch-based indexer, single-node.
- `wazuh.manager`  (172.25.0.30) — the manager. Port `1514` (agent enrollment)
  and `55000` (manager API) are mapped to the host.
- `wazuh.dashboard` (172.25.0.40) — the web UI. Reachable from the host at
  `https://localhost:443`.
- `victim` (172.25.0.10) — Ubuntu 22.04 with `sshd` and `wazuh-agent` 4.14.5.
  Port `22` is mapped to the host as `2222`.

All four share a single user-defined bridge network (`lab-net`,
`172.25.0.0/24`). The host is the attacker: run hydra against
`localhost:2222` (or the victim IP if you want logs to look like an external
hit).

## Prerequisites

- Docker Engine 24+ and Docker Compose v2.
- ~6 GB of free RAM. The Wazuh containers are the heavy ones; the victim is
  under 300 MB.
- ~5 GB of free disk for images and persistent volumes. The first pull is
  the slow one; after that, restarts are seconds.
- (Roadmap items #8+) `kubectl` and `minikube` for the Kubernetes leg.
  They are not required for the SSH brute-force detection covered by this
  README. Install notes live in
  [`bitacora/2026-08-04-host-setup-docker-k8s.md`](../bitacora/2026-08-04-host-setup-docker-k8s.md).

## First-time setup

```bash
cd setup
docker compose up -d
```

Then watch the boot order:

```bash
docker compose ps
docker compose logs -f wazuh.indexer   # wait until healthcheck is 'healthy'
docker compose logs -f wazuh.manager   # then this
docker compose logs -f wazuh.dashboard # finally this
```

Open `https://localhost:443` in a browser. Default credentials: `admin` /
`SecretPassword` (yes, the dashboard and the indexer share that password; it
is set in `docker-compose.yml` and is meant to be replaced before any
non-local use).

## Generating the manager SSL certificates

The compose file mounts `./conf/manager_ssl_certs/` into the manager and the
dashboard. The directory must contain:

- `root-ca.pem`
- `filebeat.pem`
- `filebeat.key`

These are produced by the official Wazuh tool:

```bash
docker run --rm \
  -v "$(pwd)/conf/manager_ssl_certs:/certificates" \
  wazuh/wazuh-indexer:4.14.5 \
  bash -c '
    /usr/share/wazuh-indexer/bin/indexer-certgen.sh -n "admin" -d /certificates
  '
```

If you skip this step, the manager will not be able to talk to the indexer
and you will see `503` from the API. The error is loud, not silent.

## Verifying the agent is enrolled

```bash
docker compose exec victim /var/ossec/bin/wazuh-agentd -t
docker compose logs victim | grep -E 'Connected|Active'
```

In the dashboard, **Server management → Endpoints summary** should show the
victim container as `Active`. If it shows `Pending` or `Never connected`,
the most common cause is a manager restart that wiped the agent key; in that
case:

```bash
docker compose exec victim rm -f /var/ossec/data/queue/agentd/*
docker compose restart victim
```

## Generating SSH failures by hand

Before running hydra, prove the rule chain works:

```bash
ssh -p 2222 -o StrictHostKeyChecking=no wronguser@localhost
# repeat 6 times with different usernames
```

You should see rule `5710` (sshd authentication failure) firing in the
dashboard. After the threshold, rule `5712` (multiple authentication
failures) fires too. If neither fires, the agent is not reading
`/var/log/auth.log` — check `localfile` in `conf/ossec.conf` and
`docker compose logs victim`.

## Known gotchas

- **`restart: unless-stopped` is not the same as a healthy boot.**
  The indexer takes ~2 minutes to reach `green`. If you stop the stack and
  start it again quickly, the manager may start before the indexer is
  ready. The fix is already in the compose: `depends_on: service_healthy`.
  Do not remove that block.
- **`init: true` on the victim is required.** Without it, `sshd` and
  `wazuh-agentd` are PID 1 inside the container and get reaped by the
  kernel on the first signal. The Dockerfile uses `dumb-init` as the entry
  point for the same reason.
- **The first `docker compose up` is slow.** Three Wazuh images plus the
  custom victim build. Subsequent runs are fast.
- **`docker compose down` keeps the data.** Alerts, custom rules and the
  agent key all survive a plain `down`. Use `down -v` to wipe everything
  and start from a clean state.
- **Docker Desktop on Linux is supported but not ideal.** minikube's first
  `start` against Docker Desktop is slower than against Docker Engine, and
  minikube prints a warning recommending Docker Engine instead. The lab
  works, just budget extra minutes for the first pull.

## Reset

```bash
docker compose down        # stop, keep volumes
docker compose down -v     # stop, wipe volumes (full reset)
docker compose up -d --build   # after -v, rebuilds the victim image too
```

## What is intentionally not in this README

- Custom detection rules. Those live in
  [`docs/writeups/`](../docs/writeups/) once they exist.
- The K8s component (roadmap item #8). That will use Kustomize + minikube
  and is scheduled for after items #1 and #5 ship. See
  [`docs/decisions/0002-migrate-vms-to-docker.md`](../docs/decisions/0002-migrate-vms-to-docker.md).
- Anything that would require a credential, IP or hostname from the
  operator's environment. If you find yourself wanting to put one in, push
  back and use a placeholder.
