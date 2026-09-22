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
The dashboard is reachable only through an SSH tunnel. No endpoints are
connected yet, and no alert has been generated or investigated.

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

## Planned Next Steps

- Connect `ubuntu-mgmt` as the first Wazuh agent (Linux endpoint)
- Connect a Windows endpoint with Sysmon
- Generate safe test events, investigate the resulting alerts, and write an
  incident report
- Map lab evidence to security controls
