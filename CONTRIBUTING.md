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
MOLECULE_DISTRO=debian13 uv run molecule test  # default
```

### Container engine

The scenario uses the `containers` driver, which runs the tests with Podman if it is installed
and with Docker otherwise. To choose the engine explicitly:

```shell
MOLECULE_CONTAINERS_BACKEND=docker uv run molecule test
MOLECULE_CONTAINERS_BACKEND=podman uv run molecule test
```

- **Docker:** the Docker daemon must be running and the `docker` CLI must be on the `PATH`.
- **Podman:** on macOS and Windows, a Podman machine must be running (`podman machine start`).
  The test container is privileged and runs systemd, which is tested with a rootful Podman
  machine.

### Isolating installed collections

By default, Molecule installs the collection under test and the scenario requirements into
`~/.ansible/collections`. To keep them inside the repository, set `ANSIBLE_HOME`:

```shell
export ANSIBLE_HOME="$PWD/.ansible"
```

It must be set as an environment variable, not in `ansible.cfg`, because Molecule reads it to
decide where to install the collection.

### Development loop

```shell
uv run molecule converge  # create the container and apply the role
uv run molecule login     # open a shell in the container
uv run molecule verify    # run the checks in verify.yml
uv run molecule destroy   # remove the container
```
