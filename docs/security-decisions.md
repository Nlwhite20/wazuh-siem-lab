# Security Decisions

> Status: reflects the deployment as of 2026-10-06. Between 2026-09-28 and
> 2026-10-06 the dashboard binding below was not in effect; see
> [Finding 001](finding-001-dashboard-bind-drift.md).

This lab makes a few deliberate trade-offs. Each is recorded here with its
reasoning, so the choice is documented rather than silent, in line with
change-control practice.

## Port exposure

| Port | Decision | Reasoning |
| --- | --- | --- |
| 443 (dashboard), 55000 (manager API), 9200 (indexer API) | Bound to `127.0.0.1` only, reached through an SSH tunnel | These are administrative and data-plane interfaces with no reason to be reachable from the network. Docker publishes ports around the host firewall, so the bind address is the real control, not UFW. |
| 1514, 1515 (agent event and enrollment) | Left open on the VM's Shared Network interface | Agents (`ubuntu-mgmt`, and later a Windows endpoint) need to reach these to report in. UTM's Shared Network is NAT-only and not reachable from the home LAN or the internet, so this is an accepted risk scoped to the isolated lab network. |
| 514/udp (syslog) | Removed from the compose file entirely | Not used by any planned lab traffic. |

## Credential rotation

| Account | Status | Reasoning |
| --- | --- | --- |
| `admin` (indexer) | Rotated | Default is publicly known; this is the dashboard login and the indexer's administrator. |
| `wazuh-wui` (manager API) | Rotated | Default is publicly known; used for the dashboard-to-manager connection. |
| `kibanaserver` (indexer) | **Deferred, left on default** | Internal service account used only for the dashboard's connection to the indexer, over the private Docker network (`single-node_default`), never published to the VM's network interface or the internet. Wazuh's own procedure warns that changing more than one indexer account's password in a single pass increases the chance of a lockout, and this account carries materially lower risk than `admin`. Tracked as an open item; rotating it is the next indexer-password change to make. |

## TLS certificate hostname verification

`securityadmin.sh`, the tool used to apply indexer configuration changes, is
run with `-nhnv` (disable hostname verification) because the indexer's
certificate is issued for `CN=wazuh.indexer` rather than `127.0.0.1`. This
is scoped narrowly:

- It applies only to this one internal administration command, executed from
  inside the VM, over the private Docker network.
- It does not change how the dashboard, the manager, or any external client
  connects; those all use the certificate's real hostname internally
  (`wazuh.indexer`) via Docker's DNS, and TLS is still in effect end to end.
- The alternative (regenerating certificates with `127.0.0.1` as a subject
  alternative name, or always invoking the tool by container hostname) was
  considered and set aside as unnecessary complexity for a single-node lab.

## Secrets handling

- No real password, certificate, or private key is committed to this
  repository. The live `docker-compose.yml` (which now contains real
  passwords) and the generated certificate directory
  (`config/wazuh_indexer_ssl_certs/`) are git-ignored and exist only on the
  VM.
- `docker-compose.yml` is locked to `chmod 600` on the VM after the password
  edit, so only the owning user can read it.
- New passwords were entered at hidden shell prompts (`read -rs`) and cleared
  from the shell environment (`unset`) once no longer needed, rather than
  passed as plain command-line arguments (which would be visible in the
  process list and shell history).
- A sanitized template of `docker-compose.yml`, with placeholder values in
  place of real passwords, is kept in `config-templates/` for reference.

## Open items

1. Rotate the `kibanaserver` password.
2. One agent (`soc-endpoint-01`) has enrolled through `1514`/`1515`; it has
   been Disconnected since 2026-09-28. Re-review the ports once endpoints
   are reconnected.
3. No firewall rule exists on the VM restricting which addresses can reach
   1514/1515 beyond UTM's own network isolation. Acceptable for a single-user
   home lab; would need tightening (or a Host Only network) in a
   multi-tenant setting.
4. Nothing checks the running port bindings against this document
   automatically. Run `ss -ltnH` after every change (see Finding 001).

Full list with ratings: [Risk register](risk-register.md).
