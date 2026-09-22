# Password Rotation Procedure (Wazuh 4.14.7, Docker single-node)

> Status: **Completed and verified** on this lab's `wazuh-siem` VM, 2026-09-22.

Wazuh's `wazuh-docker` repository ships with public default credentials for
three accounts (`admin` and `kibanaserver` on the indexer, `wazuh-wui` on the
manager's API). Anyone can find these values in the public repository, so a
freshly deployed stack is not safe to use until they are rotated. This
document is a worked procedure for this specific version, including two
fixes this release needed that are not in Wazuh's own published steps at the
time this was run.

## Accounts changed

| Account | Where it lives | Rotated? |
| --- | --- | --- |
| `admin` | Indexer (dashboard login, indexer administration) | Yes |
| `wazuh-wui` | Manager API, and separately the dashboard's own connection settings | Yes |
| `kibanaserver` | Internal service account, dashboard-to-indexer only, never reachable outside the Docker network | Deferred (see [Security Decisions](security-decisions.md)) |

## Summary of steps

1. Generated two new random passwords (20+ characters, letters, digits, and
   only `-`/`.` as symbols to avoid breaking YAML or shell parsing) and saved
   them in a password manager. Never typed into chat or committed anywhere.
2. Backed up `docker-compose.yml` and `config/wazuh_indexer/internal_users.yml`
   before editing either.
3. Stopped the stack (`docker compose down`, no `-v`, so named volumes and
   their data were preserved).
4. Loaded both new passwords into VM shell variables using `read -rs` (hidden
   input), never as literal command-line arguments.
5. Generated a bcrypt hash of the new `admin` password using Wazuh's own
   `hash.sh` tool, run inside a throwaway `wazuh-indexer` container.
6. Wrote that hash into `internal_users.yml`, replacing only the `admin`
   user's `hash:` field, verified by an exact-match assertion in the
   replacement script (fails loudly rather than silently doing nothing).
7. Wrote both new passwords into `docker-compose.yml`, replacing the
   `INDEXER_PASSWORD` (2 occurrences) and `API_PASSWORD` (2 occurrences)
   default values. Verified exactly 4 lines changed and that
   `docker compose config -q` still validated the file. Locked the file to
   `chmod 600`.
8. Cleared all password variables from the shell (`unset`).
9. Started the stack again (`docker compose up -d`).
10. Applied the new `internal_users.yml` to the running indexer using
    `securityadmin.sh` (see **Fixes needed**, below).
11. Restarted the stack (`docker compose restart`).
12. Verified the indexer rejects the old default `admin` password (HTTP 401)
    and accepts the new one (HTTP 200).
13. Discovered that the dashboard's *own* copy of the `wazuh-wui` credentials,
    in `config/wazuh_dashboard/wazuh.yml`, is separate from both the compose
    file and the manager's account, and still held the old default. Edited
    only its `password:` field (backed up first), restarted the dashboard
    container, and confirmed the "Check API connection" indicator turned
    green with no further errors.

## Fixes this release needed (not in Wazuh's published steps)

**1. `which: command not found` inside the indexer container.**
`securityadmin.sh` uses `which` internally to locate Java, and the container
image does not include that utility. The script fails silently, printing an
incomplete warning line and exiting without ever attempting a connection.

*Fix:* export `OPENSEARCH_JAVA_HOME=/usr/share/wazuh-indexer/jdk` before
running the tool, which the script checks first and prefers over the broken
`which` lookup.

**2. `-v` is not a recognized flag on this tool version.**
Wazuh's documented command ends in `-v` (verbose). This version's
`securityadmin.sh` rejects it outright with a usage error. *Fix:* omit `-v`.

**3. Certificate hostname mismatch when connecting to `127.0.0.1`.**
The indexer's TLS certificate is issued for `CN=wazuh.indexer`, not for
`127.0.0.1`. Connecting to the loopback address (as documented) fails Java's
strict hostname verification. *Fix:* add `-nhnv`
(`--disable-host-name-verification`) to the `securityadmin.sh` command. This
only affects this one internal administration call from inside the VM; it
does not disable TLS or weaken any externally reachable connection.

**4. The dashboard keeps its own separate copy of the `wazuh-wui` password.**
Not a bug, but an undocumented gap in the "change only two places" framing:
`config/wazuh_dashboard/wazuh.yml` has its own `password:` field for this
account, and it does not automatically follow a `docker-compose.yml` edit.
It must be changed and the dashboard container restarted separately.

## Verification evidence

- `curl -u admin:<old default> https://127.0.0.1:9200/` returns HTTP 401.
- `curl -u admin:<new password> https://127.0.0.1:9200/` returns HTTP 200.
- Dashboard login with the new `admin` password succeeds.
- Dashboard's "Check API connection" indicator is green, with no
  `[API connection] No API available to connect` error.
