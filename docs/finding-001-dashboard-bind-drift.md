# Finding 001: Dashboard port binding drifted from the documented baseline

| Field | Value |
| --- | --- |
| Found | 2026-10-06, during a repo-vs-VM reconciliation (all times UTC) |
| Fixed | 2026-10-06 02:34 |
| Asset | `wazuh-siem` VM, Wazuh dashboard (port 443) |
| Severity (lab) | Low. The VM is on UTM's NAT Shared Network, reachable from the Mac but not from the home LAN or the internet. The control failure itself (documented control not in place) is the main issue |
| Controls | NIST 800-53 CM-2 (baseline), CM-3 (change control), CM-6 (configuration settings), CA-7 (continuous monitoring) |

## What the repo said

README, [architecture](architecture.md), [security decisions](security-decisions.md)
and the compose template all state that the dashboard (443), manager API
(55000) and indexer API (9200) are bound to `127.0.0.1` only.

## What the VM showed

Read-only checks over SSH:

```
$ ss -ltnH | awk '{print $4}' | sort -u
0.0.0.0:443          <- dashboard on all interfaces
127.0.0.1:55000
127.0.0.1:9200
0.0.0.0:1514, 0.0.0.0:1515 (agent ports, accepted risk)
```

Comparing the pre-password-change backup (2026-09-21) with the live file,
with every line containing "PASSWORD" filtered out before printing:

```
75c75
<       - "127.0.0.1:443:5601"
---
>       - "443:5601"
```

That was the only non-password change. The manager API and indexer API
bindings, and the removal of 514/udp, were still in place.

## Timeline

| Time (UTC) | Event | Source |
| --- | --- | --- |
| 2026-09-21 | Dashboard, API and indexer bound to `127.0.0.1` | `docker-compose.yml.pre-pw` backup; deployment log |
| 2026-09-22 | Default passwords rotated (`admin`, `wazuh-wui`) | Deployment log |
| 2026-09-28 01:50 | Agent `soc-endpoint-01` (ID 002) runs its first file-integrity scan | `agent_control -i 002` |
| 2026-09-28 02:08 | Live `docker-compose.yml` last modified | File timestamp |
| 2026-09-28 02:18 | Agent's last keep-alive; it has been Disconnected since | `agent_control -i 002` |
| 2026-10-06 02:2x | Drift found | `ss`, filtered `diff` |
| 2026-10-06 02:34 | Fixed and verified | See below |

## Root cause

Not determined. The change was not recorded anywhere. Its timing (between
the agent's enrollment and its last keep-alive) suggests it was made during
the agent setup session, possibly to open the dashboard without the SSH
tunnel. The underlying weakness is process, not technology: a change was made
without a deployment-log entry, and nothing checks the running bindings
against the documented baseline. UFW cannot catch this, because Docker
publishes ports around it.

## Fix

1. Backed up the live file (`docker-compose.yml.pre-rebind`, permissions kept at 600).
2. Replaced line 75 with `"127.0.0.1:443:5601"`, guarded by an exact-match check
   (the command would stop if the line was not found exactly once).
3. `docker compose config -q`, then `docker compose up -d wazuh.dashboard`.
   Only the dashboard was recreated; the manager and indexer kept running.

Rollback: copy the backup back and run the same `up -d` command.

## Verification (2026-10-06 02:35)

| Check | Result |
| --- | --- |
| Listening sockets | `127.0.0.1:443`, `127.0.0.1:55000`, `127.0.0.1:9200`; agent ports 1514/1515 unchanged |
| Containers | All three Up; manager and indexer not restarted |
| Dashboard from inside the VM (`https://127.0.0.1:443/`) | HTTP 302 (redirect to login) |
| Dashboard directly from the Mac (`https://<LAB_VM_IP>/`) | No connection (curl `000`) |

## Repeatable check

Run on the VM after any change, and before trusting the documentation:

```
ss -ltnH | awk '{print $4}' | sort -u
```

Expected: 443, 55000 and 9200 appear only as `127.0.0.1`.

## Lessons

- A documented control is a claim until the running system is checked.
- Record every change the same day, including "temporary" ones.
- Diff configuration files with secret lines filtered out, so evidence can be shared safely.
