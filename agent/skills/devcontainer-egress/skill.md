---
name: devcontainer-egress
description: Manage the devcontainer-egress allowlist proxy. Use when a network request is blocked by Squid, the user asks to add or remove a host from the egress allowlist, or you need to inspect what the proxy is allowing or denying. Provides commands to add domains, reload Squid, view logs, and verify enforcement.
---

# devcontainer-egress

This machine runs a Squid forward proxy that allowlists outbound HTTP/HTTPS by
destination hostname. Containers on the `egress-agents` Docker network use
`HTTPS_PROXY=http://squid:3128`; anything not allowlisted is refused with HTTP
403, and direct egress (raw sockets) has no route to the public internet.

The user-managed allowlist is at `~/.config/devcontainer-egress/allowlist.txt`.
The `egress` command (in `~/devcontainer-egress/bin/`) is the management
interface.

## Where am I running?

The `egress` command **runs on the host**, not inside a sandboxed container.

- **On the host**: `egress` is on `PATH` (or callable as
  `~/devcontainer-egress/bin/egress`). Run commands directly.
- **Inside a sandboxed container**: `HTTPS_PROXY=http://squid:3128` is set and
  there is no `egress` binary. Ask the user to run commands on their host.

To detect: `command -v egress` succeeds → host. `$HTTPS_PROXY` contains
`squid:3128` and no `egress` on PATH → inside sandbox.

## Common commands

```sh
egress add <domain> [<domain>...]   # Append host(s) to the allowlist and reload.
                                    # Auto-prepends a leading dot for subdomain
                                    # match. Strips https:// prefix and paths.
                                    # Examples:
                                    #   egress add slack.com
                                    #   egress add linear.app notion.com
                                    #   egress add https://api.example.com/v1

egress logs                         # Tail the Squid access log live.
                                    # TCP_TUNNEL/200 = allowed.
                                    # TCP_DENIED/403 = blocked.

egress edit                         # Open the allowlist in $EDITOR
                                    # (use for removals or larger edits).
egress reload                       # Apply allowlist changes
                                    # (squid -k reconfigure).

egress test [allowed_host] [denied_host]   # Smoke-test allow/deny/bypass.
egress status                              # Show whether the proxy is up.
egress config                              # Print the allowlist path.
```

## Diagnosing a blocked request

1. Reproduce the failure with the exact command that failed.
2. In another terminal, run `egress logs`.
3. Look for `TCP_DENIED` lines — the host being blocked is on each line.
4. If you're confident the host is legitimate and the user wants it allowed:
   `egress add <hostname>`. Otherwise surface the denial to the user and let
   them decide.
5. Retry the original command.

If the failing request produces no log entry at all, the tool ignored
`HTTPS_PROXY` and tried a direct connection — the `--internal` Docker network
refuses it. Fix the tool's proxy handling; do not try to bypass the proxy.

## When inside a sandboxed container

You cannot run `egress` from inside the sandbox. On a blocked request:

1. Identify the exact hostname that's being denied (from the error message
   you got, or by asking the user to check `egress logs`).
2. Tell the user: "Request to `<hostname>` was denied by the egress proxy.
   Run `egress add <hostname>` on your host if you want it allowed."
3. Wait for confirmation before retrying.

## Things not to do

- Don't disable or override `HTTPS_PROXY` to "work around" a block — the
  proxy is intentional.
- Don't add wildcards or broad TLDs (e.g. `.com`, `.io`) — defeats the
  allowlist.
- Don't preemptively add hosts the user hasn't asked for. Add as you go;
  the audit log is part of the value.
- Don't run `egress down` / `egress restart` to recover from a denial —
  those are lifecycle commands, not fix-its.
