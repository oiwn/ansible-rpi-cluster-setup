# Agent Notes

Contract for AI coding agents working in this repo. Human-facing documentation
lives in [README.md](README.md) and [specs/overview.md](specs/overview.md);
build history is in [CHANGELOG.md](CHANGELOG.md).

## Privacy rule (IMPORTANT)

This repository is public. NEVER commit cluster-external data: MAC addresses,
home-LAN IPs, WiFi SSIDs or passwords, hardware serials, real hostnames, or
anyone's public keys.

Real values live ONLY in `inventory/hosts.yml`, which is gitignored on purpose.
`inventory/hosts.example.yml` is the committed template and must contain
placeholders only.

**Playbooks take such values from inventory hostvars and never hardcode them.**
Use `inventory_hostname`, `groups['workers']`, `hostvars[...]`, `ansible_host`,
`cluster_ip`, `mac_address` — not literals. Before committing, check with:

```sh
gitleaks git --verbose
grep -rnE '([0-9a-fA-F]{2}:){5}[0-9a-fA-F]{2}' playbooks/ inventory/
```

## Inventory

- `main` — one host: the Pi 5 gateway and k3s server. Reached over the home LAN
  via `ansible_host`.
- `workers` — the two Pi 4Bs on the cluster LAN (`10.0.0.2`, `10.0.0.3`), eth0
  only. Not routable from the home LAN; reached via ProxyJump through the
  gateway, configured as `ansible_ssh_common_args` in the inventory.
- `cluster` — convenience group covering both.

Any new variable added to the inventory must also be added, with a placeholder
and a comment naming its consumer, to `inventory/hosts.example.yml`.

## Playbooks

Numbered, run in order, each gated on the previous; `99` is read-only
diagnostics. `03` does not exist — it was an SSH-key playbook, dropped once
ProxyJump covered every access path. Do not renumber to close the gap.

Conventions to follow when editing:

- Every playbook ends with a verification block: a `changed_when: false` shell
  that gathers state, then a `debug` that prints it.
- Collections in use are declared in `requirements.yml`. Add new ones there.
- Module calls use fully-qualified names (`ansible.builtin.*`,
  `community.general.*`, `ansible.posix.*`).

<!-- BEGIN specdev -->
## specdev

This project uses **specdev** (specification-driven development). Load the
specdev skill when starting a session, continuing from specs, or picking up a
task. Always read `specs/overview.md` and `specs/ctx.md` before coding.
<!-- END specdev -->

