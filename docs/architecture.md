# Network & Architecture

> Status: **Deployed.** Reflects the running stack on the `wazuh-siem` VM.
> Uses placeholder addresses only; no real host or network details are
> recorded in this repo.

## Mac to VM access path

The Wazuh dashboard is published on the VM's loopback address only. The Mac
reaches it through an SSH tunnel, so no dashboard, API or indexer port is
open on the VM's network interface.

```mermaid
flowchart LR
    subgraph Mac["Mac host (Apple silicon, 32 GB unified memory)"]
        Browser["Browser: https://localhost:8443"]
        SSHc["SSH client, local port forward 8443 -> 443"]
        Browser --> SSHc
    end

    Net["UTM Shared Network (NAT, DHCP)"]

    subgraph VM["wazuh-siem: Ubuntu Server 24.04 ARM64 VM<br/>4 vCPU / 10 GB RAM / 96 GB disk"]
        direction TB
        SSHd["sshd :22 (UFW allows OpenSSH only)"]
        Loop["VM loopback 127.0.0.1"]
        Dash["Wazuh dashboard :443"]
        API["Wazuh manager API :55000"]
        Idx["Wazuh indexer :9200"]
        SSHd --> Loop
        Loop --> Dash
        Loop --> API
        Loop --> Idx
    end

    SSHc -->|"SSH (VM address recorded privately, not in Git)"| Net
    Net --> SSHd
```

No Bridged adapter exists. The VM is not reachable from the wider home LAN.
Docker published ports bypass UFW, so the safeguard is the `127.0.0.1` bind
address in `docker-compose.yml`, not a firewall rule.

## Docker Compose network (inside the VM)

```mermaid
flowchart TB
    subgraph net["Docker network: single-node_default (bridge)"]
        Idx["wazuh.indexer :9200"]
        Mgr["wazuh.manager :1514, 1515, 55000"]
        Dash["wazuh.dashboard :443->5601"]

        Mgr -->|filebeat, TLS| Idx
        Dash -->|query, TLS| Idx
        Dash -->|API calls, TLS| Mgr
    end

    Loopback["VM loopback 127.0.0.1"]
    LAN["UTM Shared Network"]

    Loopback -->|"published :443"| Dash
    Loopback -->|"published :55000"| Mgr
    Loopback -->|"published :9200"| Idx
    LAN -->|"published :1514, :1515 (agent enrollment/comms)"| Mgr
```

## Port exposure summary

| Port | Purpose | Binding |
| --- | --- | --- |
| 443 | Dashboard (web UI) | `127.0.0.1` only |
| 55000 | Manager API | `127.0.0.1` only |
| 9200 | Indexer API | `127.0.0.1` only |
| 1514, 1515 | Agent event and enrollment ports | Open on the VM's Shared Network interface (accepted risk, see [Security Decisions](security-decisions.md)) |
| 514/udp | Syslog input | Removed; not used in this lab |

## Known limitation

Certificate hostname verification is disabled (`-nhnv`) for the internal
`securityadmin.sh` administration step, because the indexer's certificate is
issued for the hostname `wazuh.indexer`, not for `127.0.0.1`. This only
affects that one internal maintenance command, run from inside the VM over
the private Docker network; it does not weaken the TLS between containers or
between the browser and the dashboard.
