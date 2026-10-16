# Ansible Collection max_pfeiffer.smallstep

[![CI](https://github.com/max-pfeiffer/ansible-collection-smallstep/actions/workflows/ci.yml/badge.svg)](https://github.com/max-pfeiffer/ansible-collection-smallstep/actions/workflows/ci.yml)

Ansible collection to run a [Smallstep step-ca](https://smallstep.com/docs/step-ca/) certificate
authority as infrastructure as code.

Design principles:

- **Offline root CA:** the collection deploys an existing intermediate CA and never handles the root CA key.
- **Declarative configuration:** `ca.json` is rendered completely from variables.
- **Official packages:** `step-ca` and `step-cli` are installed in pinned versions from the official
  Smallstep APT repository, which covers amd64, arm64 and armhf (Raspberry Pi).
- **Hardened service:** systemd sandboxing as recommended by Smallstep.
- **Tested on amd64 and arm64** with Molecule on Debian 12 and 13.

## Roles

| Role | Description |
|---|---|
| [`max_pfeiffer.smallstep.step_ca`](roles/step_ca/README.md) | Install and manage a step-ca server |

## Installation

```shell
ansible-galaxy collection install max_pfeiffer.smallstep
```

Or in `requirements.yml`:

```yaml
collections:
  - name: max_pfeiffer.smallstep
    version: ">=0.1.0"
```

## Development

The development environment is managed with [uv](https://docs.astral.sh/uv/). Molecule tests
run with Docker or Podman, see [CONTRIBUTING.md](CONTRIBUTING.md#testing).

```shell
uv sync
uv run ansible-lint
MOLECULE_DISTRO=debian13 uv run molecule test                        # Docker
MOLECULE_DISTRO=debian13 uv run molecule test --scenario-name podman  # Podman
```

## Releasing

1. Update `version` in `galaxy.yml`.
2. Create a GitHub release with the tag `v<version>`.
3. The release workflow publishes the collection to Ansible Galaxy. It needs the repository secret
   `ANSIBLE_GALAXY_API_KEY` in the GitHub environment `release`.
