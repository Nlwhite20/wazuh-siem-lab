# SOC Lab: Wazuh SIEM (wazuh-siem)

## Purpose

An authorized personal home-lab project that demonstrates SIEM deployment,
Docker-based service hardening, log collection, alert triage, and incident
documentation using Wazuh.

## Status

The Wazuh single-node stack (manager, indexer, dashboard) is deployed on a
dedicated ARM64 Ubuntu 24.04 VM under Docker Compose. All three services are
running, the dashboard connects successfully to both the manager API and the
indexer, and every default password has been rotated except one deferred
internal account (see [Security Decisions](docs/security-decisions.md)).
The dashboard is reachable only through an SSH tunnel.

One Linux endpoint, `soc-endpoint-01`, enrolled as an agent on 2026-09-28
and has been Disconnected since that day. No alert has been generated or
investigated yet.

A reconciliation on 2026-10-06 found that the dashboard binding had drifted
from `127.0.0.1` to all interfaces through an unrecorded change. It was
fixed and verified the same day: [Finding 001](docs/finding-001-dashboard-bind-drift.md).

## AI Collaboration and Human Validation

The 2026-10-06 reconciliation was done with Claude as an assistant. Claude
proposed read-only checks, the fix and first drafts of documentation; I ran
every command on the VM and approved the one configuration change and every
commit myself. The rules the assistant works under are in
[CLAUDE.md](CLAUDE.md) and [AGENTS.md](AGENTS.md). The earlier build entries
(2026-09-21 to 2026-09-28) do not record which steps were AI-assisted.

| What I asked for | What the AI proposed | What I checked or changed | Result |
| --- | --- | --- | --- |
| Bring the repo up to date | Update the docs from notes | Checked the live VM first: containers, listening ports, agent list | Found the dashboard listening on all interfaces, contrary to the docs (Finding 001) |
| Show what changed in the compose file | A plain `diff` | Filtered every password line out before printing | Evidence showed the single binding change without exposing secrets |
| Fix the binding | Edit, then restart the stack | Required a backup, an exact-match guard, recreating only the dashboard, and checks from both the VM and the Mac | Fixed with the manager and indexer left running |

Limitations: single-operator lab, so review is not independent. Official
documentation and observed results settle technical questions.

## Stack

- Ubuntu Server 24.04 LTS ARM64 virtual machine in UTM (Apple Virtualization)
- Docker Compose
- Wazuh manager, indexer and dashboard (v4.14.7), from Wazuh's official
  `wazuh-docker` repository, single-node layout

## Scope

Only localhost and explicitly authorized personal lab virtual machines are
monitored. No public systems, employer systems, third-party networks, or
production data are in scope.

## Security Principles

- Keep the dashboard, indexer API and manager API off the public internet;
  reach them only through an SSH tunnel bound to `127.0.0.1`.
- Rotate every default credential shipped in the public `wazuh-docker`
  repository before treating the lab as usable.
- Never commit real passwords, certificates or private keys. The live
  `docker-compose.yml` and generated certificates stay on the VM only; this
  repo holds a sanitized template instead (see `config-templates/`).
- Publish sanitized documentation and screenshots only.

## Documentation

- [Architecture](docs/architecture.md): network path, Docker topology, port exposure
- [Deployment log](docs/deployment-log.md): what was built, in order, with verification
- [Password rotation procedure](docs/password-change-procedure.md): the exact
  steps used, including two undocumented fixes this Wazuh release needed
- [Security decisions](docs/security-decisions.md): what was hardened, what
  was deferred, and why
- [Risk register](docs/risk-register.md): open, mitigated and accepted risks
- [Finding 001](docs/finding-001-dashboard-bind-drift.md): dashboard binding drift, found and fixed

## Planned Next Steps

- Reconnect `soc-endpoint-01` and confirm events arrive
- Rotate the `kibanaserver` password and apply pending OS updates
- Connect a Windows endpoint with Sysmon
- Generate safe test events, investigate the resulting alerts, and write an
  incident report
- Map lab evidence to security controls (started in Finding 001: CM-2, CM-3, CM-6, CA-7)
