# cstor-dist

This project exposes [`containers-storage`](https://github.com/containers/storage) (as used by [podman](https://github.com/containers/podman)) via the standard [OCI distribution spec](https://github.com/opencontainers/distribution-spec) (also known as a container registry).

The registry is read-only and provides a way to serve local container images over HTTP.

## Use Cases

- Enable local virtual machines to fetch content directly from the host's container storage
- Particularly useful with [bootc](https://github.com/bootc-dev/bootc/) for local development workflows (see below)

## Running the Service

While this project can run outside of a container, it currently requires a patched version of skopeo. Therefore, running it as a container is recommended.

A pre-built container image is available for x86_64; see below.

### Requirements

#### Storage Configuration

You must bind mount your host's container storage into `/var/lib/containers/storage` in the container:

- For rootless podman: `~/.local/share/containers/storage`
- For rootful podman: `/var/lib/containers/storage`

#### Privileges

The container requires `--privileged` mode for two reasons:
- Write access to storage for locking (this requirement will be removed in a future update)
- SELinux labeling support

#### Network Configuration

The service listens on port 8000 by default. You can map this to any desired host port.

### Example Usage

Start the registry proxy:
```bash
podman run --name regproxy --privileged --rm -d \
    -p 8000:8000 \
    -v ~/.local/share/containers/storage/:/var/lib/containers/storage \
    ghcr.io/cgwalters/cstor-dist:latest
```

## Using the Registry

**Important**: By default, the server does not use TLS. When using tools like `skopeo`, you must specify `--src-tls-verify=false`.

Example of copying an image:
```bash
skopeo copy --src-tls-verify=false \
    docker://127.0.0.1:8000/quay.io/fedora/fedora:latest \
    oci:/tmp/foo.oci
```

## Building from Source

1. Clone the repository:
```bash
git clone https://github.com/cgwalters/cstor-dist.git
cd cstor-dist
```

2. Build using podman or docker:
```bash
podman build -t cstor-dist .
```

## Integration with bootc

### Overview

While containers are typically run on the same machine where they're built when using podman/docker, [bootc](https://github.com/bootc-dev/bootc/) is commonly used in a distributed setup where you build on one machine and test on another.

This project works particularly well with [Anaconda](https://docs.fedoraproject.org/en-US/bootc/bare-metal/#_using_anaconda) on Linux host systems. You'll just
need to point your `ostreecontainer` at the cstor-dist endpoint.

### Efficient Development Workflow

This enables a quicker iteration workflow:

1. Build containers in your regular unprivileged podman storage
2. Use `bootc upgrade` to efficiently deploy changes without data transfer overhead
