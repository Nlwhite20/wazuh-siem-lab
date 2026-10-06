# Deployment Log

| Date | Change | Purpose | Verification | Result |
| --- | --- | --- | --- | --- |
| 2026-09-21 | Downloaded and checksummed Ubuntu 24.04.5 Server ARM64 installer | Verified base OS image before use | `shasum -a 256 -c` against Ubuntu's published SHA256SUMS: OK | Complete |
| 2026-09-21 | Created `wazuh-siem` VM in UTM (Apple Virtualization) | 4 vCPU, 10 GB RAM, 100 GB disk, matching Wazuh's single-node minimums | UTM VM settings panel | Complete |
| 2026-09-21 | Installed Ubuntu Server 24.04, OpenSSH enabled, no snaps selected | Base OS with remote access | Reached `wazuh-siem login:` prompt, then SSH from Mac | Complete |
| 2026-09-21 | Extended the LVM volume to use the full disk | Installer's default only allocated about half of the 100 GB disk | `df -h /` showed 96G after `lvextend -r` | Complete |
| 2026-09-21 | System updates applied | Current security patches before adding services | `apt upgrade`, no reboot required | Complete |
| 2026-09-21 | UFW firewall enabled: deny incoming, allow outgoing, SSH allowed | Baseline network hardening | `ufw status verbose` shows active, deny/allow defaults, SSH only | Complete |
| 2026-09-21 | Docker Engine installed from Docker's official apt repository | Container runtime for Wazuh | `docker run hello-world` succeeded; `docker --version` / `docker compose version` printed | Complete |
| 2026-09-21 | `nicholas` added to the `docker` group | Run Docker without `sudo` | `docker ps` succeeded after re-login, no `sudo` | Complete |
| 2026-09-21 | Confirmed `vm.max_map_count` already met Wazuh's 262144 minimum | Indexer startup requirement | `sysctl vm.max_map_count` returned 1048576 | Complete |
| 2026-09-21 | Cloned `wazuh-docker` at tag `v4.14.7` | Pin to a verified, specific release | `git ls-remote --tags` confirmed the tag before cloning | Complete |
| 2026-09-21 | Hardened `docker-compose.yml`: removed the syslog port, bound dashboard/API/indexer ports to `127.0.0.1` | Prevent Docker from publishing SIEM interfaces to the network | `diff` against the original file; `docker compose config -q` valid | Complete |
| 2026-09-21 | Generated indexer TLS certificates | Required for manager/indexer/dashboard TLS | Certificate files present in `config/wazuh_indexer_ssl_certs/` | Complete |
| 2026-09-21 | Started the Wazuh stack (manager, indexer, dashboard) | Bring the SIEM online | `docker ps` showed all three `Up`; `ss -ltn` showed loopback-only binding for 443/9200/55000 | Complete |
| 2026-09-22 | Rotated `admin` and `wazuh-wui` default passwords | Remove publicly known default credentials before real use | See [Password Rotation Procedure](password-change-procedure.md) for full detail and verification | Complete |
| 2026-09-28 | Enrolled `soc-endpoint-01` (Ubuntu ARM64, kernel 7.0) as Wazuh agent ID 002 | First endpoint telemetry | Recorded 2026-10-06 from the manager: `agent_control -i 002` shows agent v4.14.7, file-integrity scan 01:50 UTC, last keep-alive 02:18 UTC | Complete, agent Disconnected since |
| 2026-09-28 | **Unrecorded change:** dashboard binding changed from `127.0.0.1:443` to all interfaces | Not recorded | Found 2026-10-06 by `ss` and a password-filtered `diff`; file modified 02:08 UTC | Drift (see Finding 001) |
| 2026-10-06 | Reconciled repo with the running VM | Check documented controls against the live system | VM and Mac clocks agreed; read-only SSH checks of containers, ports, agents | Complete |
| 2026-10-06 | Rebound dashboard to `127.0.0.1:443`; only the dashboard container recreated | Restore the documented baseline | `ss` shows `127.0.0.1:443`; HTTP 302 from inside the VM; no connection from the Mac directly | Complete ([Finding 001](finding-001-dashboard-bind-drift.md)) |
| — | Rotate `kibanaserver` password | Close remaining default-credential gap | — | Pending |
| — | Apply 8 pending OS updates | Patch currency | — | Pending |
| — | Reconnect `soc-endpoint-01` | Restore endpoint telemetry | — | Pending |
| — | Connect a Windows endpoint with Sysmon | Second endpoint, richer telemetry | — | Pending |
| — | Generate and investigate a test alert; write incident report | SOC workflow evidence | — | Pending |
