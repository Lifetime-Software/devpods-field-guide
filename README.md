# DevPods Field Guide

This is a privacy-safe recovery guide for one portable Linux development
workstation. It deliberately contains no personal hostnames, usernames, IP
addresses, account identifiers, repository locations, or cryptographic keys.

## Instructions for an AI Assistant

Help the user operate or recover DevPods using this guide. Ask for missing
placeholders instead of guessing them.

Never:

- start a container directly with Docker Compose;
- bypass or overwrite the global active-host lease;
- run two restored copies simultaneously;
- delete persistent volumes without a verified encrypted snapshot;
- request that the user paste private keys, tokens, or recovery identities.

Use `devpods status` first. Prefer normal activation or handoff. Forced takeover
is only for a genuinely unreachable host.

## Mental Model

DevPods is one persistent Linux workstation that can move between trusted
Windows, Linux, USB, or cloud hosts. Only one host may run it at a time.

```text
trusted host
    |
    +-- DevPods container
    |     +-- persistent home
    |     +-- persistent projects
    |     +-- disposable caches
    |
    +-- encrypted snapshot <-> private GitHub Release
    +-- active-host lease   <-> private Git reference
```

Activation acquires the global lease, restores a newer encrypted snapshot,
starts the workstation, verifies health, and starts the idle monitor.

Deactivation encrypts and uploads the latest state, verifies the remote asset,
stops the workstation, and releases the lease. If synchronization cannot be
verified, normal shutdown is cancelled.

## Everyday Commands

Run these on a configured host:

```text
devpods status
devpods activate
devpods connect
devpods sync
devpods deactivate
devpods doctor
```

From any device connected to the same Tailscale network:

```text
ssh dev@devpods
```

## Starting It from a Phone

Requirements:

- the Windows host is powered on and awake;
- the host is connected to Tailscale;
- Docker Desktop is running;
- the one-time restricted phone-start installer was completed;
- the Windows GitHub CLI login captured by that installer is still valid;
- the phone is connected to Tailscale and has its authorized SSH key.

First ask the host to activate DevPods:

```text
ssh <windows-user>@<windows-host>
```

That key is restricted to the activation helper and does not open a general
Windows shell. The installer protects the host's activation credential with
Windows DPAPI because an SSH network logon cannot use the interactive desktop's
credential-manager session. When it reports ready, connect to the workstation:

```text
ssh dev@devpods
```

The two commands may be combined:

```text
ssh <windows-user>@<windows-host> && ssh dev@devpods
```

This cannot start a powered-off or sleeping PC. Wake-on-LAN, an always-on home
device, or a cloud host is required for that situation.

## What Persists

The encrypted snapshot contains:

- the Linux home directory;
- Claude and Codex account state, settings, and product-managed memory;
- shell configuration, Git configuration, and user-installed tools;
- project repositories, uncommitted work, and other workspace files;
- SSH authorization and the portable Tailscale workstation identity.

Package and build caches persist only on the current host and are not uploaded.
Project `.venv` directories are also omitted because they contain
image-specific interpreters and are recreated from dependency manifests.
Running processes and development servers stop with the container. Tmux
sessions are saved as restorable tabs, panes, layouts, working directories, and
history; Codex and Claude panes reopen their saved-conversation pickers when
tmux starts again. This is workspace reconstruction, not process-memory
checkpointing. System changes made outside persistent directories disappear on
an image rebuild; permanent base packages belong in the Dockerfile.

Git repositories still need their ordinary remotes. The encrypted snapshot
protects uncommitted workstation state but does not replace normal Git pushes.

## Moving to Another Host

On the active host:

```text
devpods handoff <target-name>
```

On the target host:

```text
devpods activate
ssh dev@devpods
```

The target needs the same encrypted-state recovery identity. Keep that identity
in a secure external location. The private GitHub snapshot is intentionally
useless without it.

## If Activation Is Blocked

Run:

```text
devpods status
```

If another host owns the lease, connect to that host and deactivate it. If that
host is truly unreachable and cannot return, use forced takeover only after
confirming it is not running:

```text
devpods takeover --force
```

Never use takeover merely to avoid a normal handoff.

## Troubleshooting Order

On a configured host:

```text
devpods status
devpods doctor
docker logs --tail 100 devpods
```

Then inspect the local DevPods monitor log and verify:

- Docker is running;
- Tailscale is connected;
- GitHub authentication can access the private repository and Releases;
- the recovery identity exists;
- the active host has not lost its lease.

For a changed SSH fingerprint, verify the expected host and fingerprint before
removing a saved entry. For `Permission denied (publickey)`, install the client
device's public key through the documented setup flow; never share its private
key.

## Automation

A lightweight repository-scoped GitHub Actions runner may run on a trusted host
outside the DevPods container. It can continue quality checks, cloud cleanup,
and publication of this guide while the workstation container is inactive.

The runner is available only while its host is awake and online. Workflow code
executes on that host, so it must remain restricted to trusted private
repositories and narrowly scoped deployment credentials.

## What to Ask the User For

An assistant may safely ask for:

- which trusted host is reachable;
- the non-secret Tailscale/MagicDNS host alias;
- the output of `devpods status` or `devpods doctor`;
- whether the PC is awake and Docker is running;
- exact error text with credentials removed.

An assistant should never ask for private SSH keys, GitHub tokens, the encrypted
snapshot recovery identity, Claude/Codex credentials, or decrypted archives.
