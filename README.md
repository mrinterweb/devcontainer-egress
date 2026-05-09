# devcontainer-egress

A hostname-allowlist egress proxy for sandboxed development environments.

Drop any container onto its `egress-agents` Docker network and outbound
traffic is forced through Squid, where only allowlisted domains are reachable.
A process that ignores `HTTPS_PROXY` and dials a raw IP cannot escape — the
network has no route to the public internet except through the proxy.

Built for running coding agents (Claude Code, Aider, Codex, …) without giving
them open internet, but agnostic to the agent and the LLM provider you choose.
Runs on macOS and Linux.

## Why

Coding agents routinely run shell commands, install packages, and call out to
LLM APIs. Without egress controls, that's an open path for an agent to exfiltrate
credentials, fetch hostile code from arbitrary hosts, or call providers you
didn't intend.

This project gives you:

- A single user-managed allowlist of hosts your agents are allowed to reach,
  stored at `~/.config/devcontainer-egress/allowlist.txt` (outside this repo).
- Enforcement via an `--internal` Docker network, so the allowlist can't be
  bypassed by ignoring proxy environment variables.
- A live audit log of every request, allowed or denied.
- One shared stack per machine. Any number of devcontainers or `docker run`
  workloads can share it.

## Requirements

- Docker (Docker Desktop, OrbStack, or Colima on macOS; distro Docker on Linux)
- Bash 3.2+ (for the `egress` wrapper script)

## Installation

```sh
git clone https://github.com/<you>/devcontainer-egress ~/devcontainer-egress
ln -s ~/devcontainer-egress/egress ~/.local/bin/egress   # or /usr/local/bin
egress up
egress test
```

On Linux, ensure your user is in the `docker` group:

```sh
sudo usermod -aG docker "$USER"   # then log out and back in
```

The Squid container is set to `restart: unless-stopped`, so once you have run
`egress up`, the proxy comes back automatically across host reboots.

## Usage

```
egress up        Start the proxy (idempotent; bootstraps allowlist on first run)
egress down      Stop and remove the proxy and its networks
egress reload    Reload Squid config without dropping connections
egress restart   Restart Squid (full container restart)
egress status    Show stack status
egress logs      Tail the Squid access log live
egress test      Smoke-test allow / deny / bypass
egress edit      Open the allowlist in $EDITOR
egress config    Print the path to the user allowlist
egress help      Show this help
```

`egress test` accepts optional hosts:

```sh
egress test api.openai.com badhost.example
```

## Connecting a devcontainer

Add the following to any project's `.devcontainer/devcontainer.json`:

```jsonc
{
  "name": "my-project",
  "build": { "dockerfile": "Dockerfile" },
  "remoteUser": "vscode",
  "runArgs": ["--network=egress-agents"],
  "containerEnv": {
    "HTTPS_PROXY": "http://squid:3128",
    "HTTP_PROXY":  "http://squid:3128",
    "NO_PROXY":    "localhost,127.0.0.1"
  }
}
```

The Dockerfile is whatever the project needs (language toolchain plus any agent
CLIs). The egress wiring is identical across projects, so any number of
devcontainers can share the same proxy and allowlist.

## Connecting a plain container

```sh
docker run --rm -it \
  --network egress-agents \
  -e HTTPS_PROXY=http://squid:3128 \
  -e HTTP_PROXY=http://squid:3128 \
  -e NO_PROXY=localhost,127.0.0.1 \
  ubuntu:24.04 bash
```

## Configuration

The allowlist is a plain-text file outside this repo:

```
~/.config/devcontainer-egress/allowlist.txt
```

(`$XDG_CONFIG_HOME/devcontainer-egress/allowlist.txt` if you have
`XDG_CONFIG_HOME` set.)

The file is created automatically on first `egress up` from
[`defaults/allowlist.txt`](./defaults/allowlist.txt). One hostname per line; a
leading dot matches all subdomains; `#` starts a comment:

```
.anthropic.com
.openai.com
.github.com
.npmjs.org
.pypi.org
```

Edit and apply:

```sh
egress edit
egress reload
```

Run `egress logs` in another terminal while debugging — it shows
`TCP_TUNNEL/200` for allowed CONNECTs and `TCP_DENIED/403` for blocked ones,
with the target host on each line.

## How it works

```
┌──────────────────────────┐       ┌──────────────┐       ┌──────────────┐
│ container                │       │ egress-squid │       │  Internet    │
│ HTTPS_PROXY=squid:3128   │ ────▶ │ (allowlist)  │ ────▶ │ (allowlisted │
│ network: egress-agents   │       │              │       │  domains)    │
└──────────────────────────┘       │ networks:    │       └──────────────┘
                                   │  egress-agents (--internal)
                                   │  egress-proxy-out
                                   └──────────────┘
```

`egress up` creates two Docker networks and a Squid container:

- `egress-agents` is `--internal` — Docker omits the NAT rules that normally
  let containers reach the public internet. The only host on this network with
  an outside path is Squid, because Squid is also a member of the second
  network, `egress-proxy-out`, which has normal connectivity.
- Containers on `egress-agents` reach Squid by DNS as `squid:3128`.
- Squid filters CONNECT requests by destination hostname. For HTTPS that
  hostname comes from the CONNECT line; for HTTP, from the request URL or Host
  header. There is no TLS interception and no CA injection.

## Limitations

- **Hostname-level only.** Squid sees the SNI / Host but not URL paths or
  request bodies inside encrypted traffic. For URL or body filtering, switch
  to a MITM proxy (e.g. mitmproxy) with a CA injected into the container.
- **Container isolation only.** All workloads share the host kernel. If your
  threat model includes kernel exploits in untrusted code, use a microVM-based
  sandbox (microsandbox, Kata Containers).
- **Build-time traffic bypasses Squid.** `RUN` steps in a project's Dockerfile
  use Docker's build network, not the runtime network. Only the running
  container is constrained.

## Troubleshooting

**A tool can't reach a domain.** Run `egress logs` and trigger the request. A
`TCP_DENIED` line means the host is missing from your allowlist — add it with
`egress edit` and `egress reload`. No log line at all means the tool ignored
`HTTPS_PROXY` and tried direct egress, which the internal network refuses by
design.

**`docker: permission denied` on Linux.** Your shell still has the pre-`usermod`
group set. Log out of the desktop session and back in, then verify with `id`.

**Squid restart loop.** Inspect `docker logs egress-squid`. The most common
cause is a syntax error in `squid.conf`.

## License

[MIT](./LICENSE)
