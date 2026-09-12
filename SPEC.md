# TIP Network Monitor, specification

Draft for review. Owner: TIP infrastructure. Updated 2026-09-11.

This repo is the outside view of the TIP network: uptime checks, status page, heartbeat, alert routing, the federation registry watcher, backups and runbooks. Metrics, dashboards and logs (Prometheus, Grafana, Loki) stay in `tip-protocol/infra/observability` because they ship with the node and every partner runs them unchanged.

Rule: if a partner would deploy it identically, it goes in the protocol repo. If it names our hosts, channels or people, it goes here.

## 1. What runs today (VP box, 52.87.175.253)

Verified from the box on 2026-09-11. The VP application (`tip-vp` container, ports 5050 and 3003) runs on the same host.

**Uptime Kuma**
- Version 2.5.3, Docker container `uptime-kuma`, image `louislam/uptime-kuma:2`, port 3001, plain HTTP, no reverse proxy.
- Port 3001 is closed at the security group. Access is by SSH tunnel only.
- One user (`theailab`). No 2FA.
- Data in Docker volume `uptime-kuma` (`kuma.db` SQLite, WAL mode). No backup.
- Retention 365 days. No status page created. Certificate expiry notification off on every monitor.

**Monitors, all 60 s, accepted status 200 to 299, all UP**

| Monitor | Type | URL | Assertion |
|---|---|---|---|
| TIP node | json-query | https://node.theailab.org/health | `$.data.consensus.halt.halted == false` |
| TIP node2 | json-query | https://node2.theailab.org/health | same |
| TIP node3 | json-query | https://node3.theailab.org/health | same |
| TIP VP | json-query | https://vp.theailab.org/health | `$.ready_for_production == true` |
| AI classifier | http | https://tipclassifier.theailab.org/health | 200 only |
| Grafana | http | https://grafana.theailab.org/login | 200 only |

**Notifications**
- One Slack notification, "Slack - tip-alerts", set as default and bound to all six monitors. Which webhook it uses is not recorded anywhere.
- Healthchecks.io: one check, period 5 minutes. Two email recipients, both still unconfirmed.

**Cron on the VP host**
- Every 5 min: ping Healthchecks, but only when root disk usage is under 90%.
- Every 15 min: `/home/ubuntu/disk-alert.sh` posts to Slack when root disk crosses 80% (warning) or 90% (critical), and on recovery. Tracks the last state in `/tmp/disk-alert-state` so it posts once per change.

**Down events recorded so far**
- AI classifier 502 on 2026-09-05 at 21:08, 21:13, 21:20 UTC and on 2026-09-08 at 19:38 UTC.
- Not recorded: the classifier's `/v1/prescan` answered 404 from Sep 5 to Sep 8 while `/health` stayed 200. No check caught it.

## 2. Put it in the repo (M1)

Nothing new to build. Commit the running state as files so it can be rebuilt without anyone's memory.

- [ ] `kuma/docker-compose.yml`: the container exactly as deployed, image pinned to 2.5.3, the named volume, port bound to `127.0.0.1:3001`.
- [ ] `kuma/README.md`: where it runs, how to tunnel in, who holds the login.
- [ ] `monitors.yml`: the six monitors from section 1 as data (name, type, url, interval, assertion, group, severity, notification).
- [ ] `heartbeat/cron.d`: both cron lines, with the ping URL and webhook read from an env file.
- [ ] `heartbeat/disk-alert.sh`: as running, webhook from env.
- [ ] `notifications.md`: the Slack notification and the Healthchecks check without secrets, plus which webhook is bound today.
- [ ] `INVENTORY.md`: every service, its URL, owner, and what the current check proves.
- [ ] `INCIDENTS.md`: seeded with the events in section 1.
- [ ] `.env.example` naming every secret, `.gitignore` covering `.env` and `kuma.db`.
- [ ] `README.md`: step-by-step setup of everything in section 1 on a fresh Ubuntu box: Docker, the Kuma compose, restoring or applying monitors, the Slack and Healthchecks wiring, the cron entries, the tunnel command, and how to verify each piece is working. Written so someone who has never seen the box can do it.
- [ ] Rotate the Kuma password (it was shared in chat) and confirm the two Healthchecks emails.

Done when: a fresh box can be brought to the section 1 state from the repo plus the secret store.

## 3. Add next, in this order

### M2, checks that catch real failures
- [ ] Node monitors assert the fields below, not only `halted`. Kuma takes one JSON expression per monitor, so use one monitor per row in a group per node (or a combined expression if preferred, decide in `monitors.yml`).

| Field on `GET /health` | Pass | Catches | Severity |
|---|---|---|---|
| `data.status` | `ok` | app failure | critical |
| `data.consensus.halt.halted` | `false` | fork halt | critical |
| `data.consensus.halt.staleMs` | under 300000 | reachable but chain not advancing | critical |
| `data.consensus.narwhal.running` | `true` | consensus loop stopped | critical |
| `data.consensus.narwhal.joinState` | `ready` | syncing or catching up | warning after 15 min |
| `data.peers.connected` | 2 or more | isolation | warning |

- [ ] Classifier function probe: every 5 min `POST /v1/prescan` with a dedicated monitoring key and a fixed 300-character text, `origin_code` OH. Pass when 200 and `probability` is a number in 0 to 1. Keep the existing 200 check as liveness. This is the check that would have caught Sep 5 to 8.
- [ ] Certificate expiry on every HTTPS monitor, notify at 21, 14 and 7 days.
- [ ] Nightly `sqlite3 .backup` of the Kuma volume to S3, 30 day retention, restore steps in `runbooks/restore-kuma.md`.
- [ ] Same job also backs up the VP app's `.env` and `server/data/` (bind mounts under `/home/ubuntu/tip-vp`), encrypted. Today the `.env` with about 40 secrets (OAuth client pairs, Gemini, Resend, VP admin key) has no copy anywhere; put those in the team secret store as well.

- [ ] Gaps found on review, add as monitors now:
  - AZ Logics node at https://tipnode.azlogics.com with the node assertions above, in a "Federation nodes" group, warning severity. It is not on chain as an endpoint yet, so the watcher (M4) cannot discover it; add by hand until then.
  - Test cluster (https://testnode.theailab.org) in its own "Test network" group, info severity, so a broken test node never pages but is visible.
  - Media storage per region: a presigned GET of one known small object in each node bucket (us-east-1, us-west-2, ap-south-1), every 15 min. Uploads and the classifier both fail silently when a bucket or its KMS key is misconfigured.
  - Metrics pipeline freshness: Grafana `GET /api/health` (database ok) and, through the Grafana datasource proxy, the Prometheus query `min(up{job="tip-federation"})` equal to 1. Catches a stopped Prometheus or a node that silently dropped out of scraping; Prometheus and Loki are not reachable from outside so this is the only way to probe them.
  - Response-time ceiling on the node and VP monitors (Kuma "max ping", 2000 ms, warning). A node answering `/health` in seconds is usually a blocked event loop.
  - Domain expiry for `theailab.org` and `azlogics.com` (Kuma domain expiry notification), 30 days.
  - A quarterly test alert to Slack and email, logged in `INCIDENTS.md`, so a dead webhook or unconfirmed recipient is found before an outage.

Done when: a stuck node and a broken prescan route both alert within 5 minutes.

### M3, status page and a heartbeat that watches Kuma
No new box and no certificate on the box: the page is Kuma's built-in status page, published through Cloudflare the same way `vp.theailab.org` reaches this host (proxied DNS, origin port rule).
- [ ] In Kuma: create the status page (slug `tip`), groups Network (node 1 to 3), Federation nodes, Services (VP, classifier, Grafana). Public read only, 90-day history, certificate expiry shown.
- [ ] Cloudflare DNS: `status.theailab.org`, proxied, pointing at the VP box. Cloudflare terminates TLS.
- [ ] Cloudflare origin rule for that hostname: destination port 3001 (same pattern as the 5050 rule for the VP app).
- [ ] Security group: open 3001 to Cloudflare's published IP ranges only.
- [ ] Cloudflare WAF rule on `status.theailab.org`: allow `/status/*`, `/api/status-page/*`, `/assets/*`, `/upload/*`; block everything else, in particular `/socket.io/*` (dashboard and login). Admin access stays over the SSH tunnel.
- [ ] Kuma settings: primary base URL `https://status.theailab.org`, trust proxy on. Enable Cloudflare "Always Online" for the hostname so a stale page is served if the box is down.
- [ ] 2FA on the Kuma login.
- [ ] Heartbeat pings Healthchecks only when disk is under 90% AND the `uptime-kuma` container is healthy. Grace 10 min. Today a stopped Kuma keeps pinging.

Done when: the page is public over HTTPS, the login is unreachable from the internet, and stopping Kuma alerts within 15 minutes.

### M4, federation registry watcher
One small container in this repo, runs every 5 minutes, read-only against the network.
- [ ] Reads `GET /v1/node/registry` from two of our nodes; a disagreement is a warning.
- [ ] Creates or updates the M2 monitors in Kuma for every registered node that has an `api_endpoint`, under "Federation nodes". A node without an endpoint (AZ Logics today) is shown as "registered, endpoint not published", not skipped.
- [ ] Compares `GET /v1/state-root` across all nodes: same round must give the same root (forked, critical); more than 2000 rounds behind is lagging (warning).
- [ ] Reads committee metrics from Prometheus: `tip_committee_member` dropping to 0 (warning), `tip_committee_participation_credits` against `tip_committee_participation_required` with a pace rule (more than 7 missed hours in the UTC day means the next rotation is lost, warn at the first hour that is true). Genesis members are reported, never alerted.
- [ ] Flags `tip_db_parity_last_ok_age_ms` flat at 0 on a Postgres node (fell back to SQLite at boot) and `tip_db_oldest_pending_write_ms` over 30 s.
- [ ] Exposes its own metrics and pushes its own Healthchecks heartbeat.

Done when: a newly registered partner node appears on the status page with no manual work, and an at-risk committee member is flagged before the rotation boundary.

### M5, alerting from the inside
- [ ] Prometheus alert rules in `tip-protocol/infra/observability` (halt, stale chain, state-root divergence, persistence lag, disk free), thresholds taken from mainnet baselines.
- [ ] Alertmanager in this repo with Slack and email receivers, routing per section 4.
- [ ] `node_exporter` on every box (the scrape job exists commented out in the prod Prometheus config).

## 4. Rules

**Routing**
| Severity | Where | Response |
|---|---|---|
| critical | Slack `#tip-alerts` plus email to on-call | 15 minutes |
| warning | Slack `#tip-alerts` | same working day |
| info | Slack thread or status page note | none |

Every alert links its runbook in `runbooks/`. Still-down critical repeats every 30 minutes. Recovery is always sent. Maintenance is declared in Kuma and noted in `INCIDENTS.md`.

**Secrets**
Never in git: Kuma password and JWT secret, Slack webhooks, Healthchecks ping URLs, classifier monitoring key, S3 credentials, `kuma.db`. They live in the team secret store and are injected from a git-ignored `.env`. Anything pasted in chat or email gets rotated.

## 5. Known debts

| Debt | Risk | Closed by |
|---|---|---|
| Kuma runs on the VP box it monitors | VP outage takes monitoring and the status page down with it | accepted for now; revisit when a second operator joins |
| Kuma over plain HTTP, no 2FA | credentials in the clear if ever exposed | M3 (Cloudflare TLS, login blocked at the edge, 2FA) |
| Heartbeat proves the host, not Kuma | Kuma can be dead while pings continue | M3 |
| Node checks only assert `halted` | stuck, isolated or non-committee nodes look green | M2 |
| Classifier check is a bare 200 | route outages invisible (Sep 5 to 8) | M2 |
| No Kuma or VP box backup | uptime history and the VP `.env` lost on disk failure | M2 |
| No Prometheus alert rules | dashboards show problems, nobody is paged | M5 |

## 6. Repository layout

```
SPEC.md                    this document
README.md                  setup guide, fresh box to the section 1 state
INVENTORY.md               services, owners, what each check proves
INCIDENTS.md               dated outage log
kuma/                      docker-compose.yml, README.md
monitors.yml               every monitor as data
scripts/apply-monitors.py  applies monitors.yml to Kuma through its API (M2)
heartbeat/                 cron.d, heartbeat.sh, disk-alert.sh
notifications.md           Slack and Healthchecks definitions, no secrets
watcher/                   registry watcher (M4)
alerting/                  alertmanager.yml (M5)
backup/                    Kuma volume backup job (M2)
runbooks/                  one file per alert
.env.example, .gitignore
```
