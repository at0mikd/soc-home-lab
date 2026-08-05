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
  `https://localhost:5601`.
- `victim` (172.25.0.10) — Ubuntu 22.04 with `sshd` and `wazuh-agent` 4.14.5.
  Port `22` is mapped to the host as `2222`.

All four share a single user-defined bridge network (`lab-net`,
`172.25.0.0/24`). The host is the attacker: run hydra against
`localhost:2222` (or the victim IP if you want logs to look like an external
hit).

## Prerequisites

- Docker Engine 24+ and Docker Compose v2.
- **~7 GB of free RAM**, and I mean *free*, not "total minus browsers".
  Measured steady-state usage on this host (Docker Desktop over linuxkit):

  | Container | RSS |
  |---|---|
  | `wazuh.indexer` | ~1.4 GB |
  | `wazuh.manager` | ~600 MB |
  | `wazuh.dashboard` | ~200 MB |
  | `victim` | ~10 MB |
  | **Stack total** | **~2.2 GB** |

  On top of that, Docker Desktop itself reserves ~3.7 GB for its VM. So the
  real floor is ~6 GB just for the lab + Docker, before the host OS and any
  browser. If you plan to open a browser alongside the lab, budget 8 GB. With
  less than that the whole machine thrashes into swap and the indexer's
  cluster events start timing out (you will see
  `process_cluster_event_timeout_exception`). Those timeouts look like a
  config bug but are just a RAM problem.
- ~6 GB of free disk for images and persistent volumes. The first pull is
  the slow one (the Wazuh images total a few GB); after that, restarts are
  seconds.
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

Open `https://localhost:5601` in a browser. Default credentials: `admin` /
`SecretPassword` (yes, the dashboard and the indexer share that password; it
is set in `docker-compose.yml` and is meant to be replaced before any
non-local use).

> **On committed credentials:** the Wazuh API password
> `MyS3cr37P450r.*-` in `conf/wazuh_dashboard/wazuh.yml` and
> `docker-compose.yml` is the **default shipped by the official
> `wazuh/wazuh-docker` image**, not a real secret. It only protects the
> manager's local API on the `lab-net` bridge and presents no exposure beyond
> the lab. It is committed on purpose so the lab is reproducible; replace it
> (and `SecretPassword`) before any non-local or shared use.

> **Why 5601 and not 443?** The dashboard's HTTPS port is published on the
> host as `5601`, not the conventional `443`. Fedora's default `firewalld`
> zone only opens ports `1025-65535`, so `443` gets silently dropped for
> LAN/remote connections (a loopback `curl` still worked, which made this
> confusing). 5601 is in the allowed range, so it just works. If you really
> want `443`, open it in the firewall first:
> `sudo firewall-cmd --permanent --add-port=443/tcp && sudo firewall-cmd --reload`.

## First-time boot: initialize OpenSearch Security (do this once)

The `wazuh/wazuh-indexer:4.14.5` image ships with the `securityadmin`
auto-init block **commented out** in its entrypoint (`/entrypoint.sh`). So on
a fresh boot the indexer comes up with OpenSearch Security uninitialized and
rejects every authenticated call with
`503 Service Unavailable: OpenSearch Security not initialized`. You must run
`securityadmin` once by hand, after the indexer is up:

```bash
docker exec -u root wazuh-indexer \
  bash -c 'JAVA_HOME=/usr/share/wazuh-indexer/jdk \
  /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
  -cd /usr/share/wazuh-indexer/config/opensearch-security \
  -cacert /usr/share/wazuh-indexer/config/certs/root-ca.pem \
  -cert /usr/share/wazuh-indexer/config/certs/admin.pem \
  -key /usr/share/wazuh-indexer/config/certs/admin-key.pem \
  -h 127.0.0.1 -p 9200 -icl -nhnv'
```

Wait for `Done with success` at the end. The cluster should then report
`green` (`curl -ksS -u admin:SecretPassword https://localhost:9200/_cluster/health`).

If the indexer data volume is fresh, you will also see a half-created
`.kibana_1` migration on the dashboard's first start (the migration times out
on a slow host). If the dashboard logs
`Another OpenSearch Dashboards instance appears to be migrating the index`,
delete the empty index and restart the dashboard:

```bash
docker exec wazuh-indexer bash -c 'curl -ksS -u admin:SecretPassword \
  -X DELETE "https://localhost:9200/.kibana_1"'
docker restart wazuh-dashboard
```

This is only needed on a first boot against an empty data volume; the
security configuration and the dashboard index persist in the named volumes
after that.

## Generating the manager SSL certificates

The compose file mounts `./conf/manager_ssl_certs/` into the manager and the
dashboard. The directory must contain:

- `root-ca.pem`
- `filebeat.pem`
- `filebeat.key`
- `admin.pem` and `admin-key.pem` (used by the indexer cluster admin)

These are produced by Wazuh's official certificate generator
(`wazuh/wazuh-certs-generator`). The node list lives in `conf/certs.yml`;
the compose wrapper in `generate-indexer-certs.yml` writes the output into
`conf/manager_ssl_certs/`.

```bash
cd setup
# First time only, on the host: raise the OpenSearch mmap limit.
# Requires root (ask the operator to run it in their own terminal).
sudo sysctl -w vm.max_map_count=262144

# Generate the certificates.
docker compose -f generate-indexer-certs.yml run --rm generator
```

If you skip this step, the manager will not be able to talk to the indexer
and you will see `503` from the API. The error is loud, not silent.

> **Note:** the certs are generated into `setup/conf/manager_ssl_certs/`,
> which is in `.gitignore`. They are environment-local and must not be
> committed. If you rotate them, delete the directory contents and rerun the
> generator.

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
- **The healthchecks in this compose expect the security plugin to be up.**
  The indexer healthcheck calls the API with credentials
  (`curl -u admin:SecretPassword`), and the dashboard one only checks that
  the HTTP port answers, because once security is on, the plain `/status`
  returns `401`, not `200`. If you ever see the indexer stuck on
  `health: starting` forever, it is usually the missing `securityadmin` step
  above, not the healthcheck itself.
- **`docker compose down` can hang while the indexer is still booting.** The
  indexer's healthcheck keeps it in `starting` for a while and Compose waits
  on it. If `down` appears to hang, `docker stop wazuh-manager wazuh-indexer
  wazuh-dashboard victim` usually finishes the job faster.
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
