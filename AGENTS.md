# SOC Lab (Wazuh SIEM) Safety Rules

This repository supports an authorized personal home-lab project only.

## Authorized Scope

- Monitor only localhost and virtual machines I own in my isolated home lab.
- Do not monitor, scan, probe, or test public systems, employer systems, school systems,
  neighbors, customer systems, or third-party networks.
- Use synthetic or sanitized evidence in GitHub documentation.

## Required Approval

Ask before:
- Running `sudo`, installing software, starting or recreating Docker containers, or changing VM settings
- Changing firewall, port bindings or network settings
- Enrolling, removing or reconfiguring Wazuh agents
- Deleting containers, volumes, agents, rules, or files
- Committing, pushing, or sending data outside the local lab

## Security

- Do not expose the Wazuh dashboard, manager API or indexer API to the public internet.
  Bind them to `127.0.0.1` and reach them through an SSH tunnel.
- Do not commit passwords, certificates, private keys, `.env` files, or raw logs.
- Never print secrets to the terminal or chat; filter password lines out of any diff.
- Every configuration change gets a row in `docs/deployment-log.md` the same day.

## Documentation

For each change or finding, document: purpose, evidence, verification, and rollback.
