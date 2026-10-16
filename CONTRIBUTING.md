# Contributing

Refer to the [Ansible community guide](https://docs.ansible.com/ansible/devel/community/index.html).

## Development environment

The development environment is managed with [uv](https://docs.astral.sh/uv/):

```shell
uv sync
uv run ansible-lint
```

## Testing

The `step_ca` role is tested with [Molecule](https://docs.ansible.com/projects/molecule/) in a
systemd enabled container. Molecule installs the collection from the working tree before every
run, so there is no need to build or install it first.

```shell
MOLECULE_DISTRO=debian12 uv run molecule test
MOLECULE_DISTRO=debian13 uv run molecule test  # default distribution
```

### Container engine

There is one scenario per container engine. Both run the same `converge.yml` and `verify.yml`,
but each uses the options its engine needs to run systemd in a container.

| Scenario | Engine | Command |
|---|---|---|
| `default` | Docker | `uv run molecule test` |
| `podman` | Podman | `uv run molecule test --scenario-name podman` |

- **Docker:** the Docker daemon must be running and the `docker` CLI must be on the `PATH`. On
  cgroup v2 hosts the container shares the host cgroup namespace (`cgroupns_mode: host`).
- **Podman:** on macOS and Windows, a Podman machine must be running (`podman machine start`).
  Podman detects the systemd command of the image and sets up cgroups itself. The test
  container is privileged and runs systemd, which needs rootful Podman. Switch a Podman
  machine to rootful mode with `podman machine stop && podman machine set --rootful &&
  podman machine start`. On Linux, run Podman as root through a wrapper, like CI does:

  ```shell
  printf '#!/bin/sh\nexec sudo podman "$@"\n' | sudo tee /usr/local/bin/sudo-podman > /dev/null
  sudo chmod 0755 /usr/local/bin/sudo-podman
  MOLECULE_PODMAN_EXECUTABLE=sudo-podman uv run molecule test --scenario-name podman
  ```

### Isolating installed collections

By default, Molecule installs the collection under test and the scenario requirements into
`~/.ansible/collections`. To keep them inside the repository, set `ANSIBLE_HOME`:

```shell
export ANSIBLE_HOME="$PWD/.ansible"
```

It must be set as an environment variable, not in `ansible.cfg`, because Molecule reads it to
decide where to install the collection.

### Development loop

Add `--scenario-name podman` to each command when using Podman.

```shell
uv run molecule converge  # create the container and apply the role
uv run molecule login     # open a shell in the container
uv run molecule verify    # run the checks in verify.yml
uv run molecule destroy   # remove the container
```
