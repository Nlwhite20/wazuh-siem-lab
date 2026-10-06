# Risk Register

Status as of 2026-10-06. Lab-scale risks; likelihood and impact are judged for a single-user home lab.

| ID | Risk | Likelihood | Impact | Status | Treatment |
| --- | --- | --- | --- | --- | --- |
| R-01 | Dashboard binding drifts from the documented `127.0.0.1` baseline | Occurred once | Medium | Mitigated 2026-10-06 | Rebound to loopback ([Finding 001](finding-001-dashboard-bind-drift.md)). Residual: no automated check; run the `ss` check after every change |
| R-02 | Changes made without a record (CM-3) | Occurred once | Medium | Open | Rule added to `CLAUDE.md`/`AGENTS.md`: every change gets a deployment-log row the same day |
| R-03 | `kibanaserver` still on its public default password | Medium | Medium | Open (deferred) | Rotate next, using the [rotation procedure](password-change-procedure.md) |
| R-04 | Agent ports 1514/1515 open on the UTM Shared Network | Low | Low | Accepted | NAT-only network, not reachable from the home LAN or internet. Revisit if the network changes |
| R-05 | Agent `soc-endpoint-01` Disconnected since 2026-09-28, so the endpoint is not monitored | High | Medium | Open | Start the endpoint VM, confirm reconnection, then run test events |
| R-06 | 8 pending OS updates on `wazuh-siem` | Medium | Medium | Open | Patch in a planned window, then re-verify bindings and agent status |
| R-07 | `securityadmin.sh` run with hostname verification disabled (`-nhnv`) | Low | Low | Accepted | Internal admin command only; see [security decisions](security-decisions.md) |
