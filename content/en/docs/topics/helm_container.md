---
title: "Running Helm in a Container"
description: "Describes how to run Helm in a container."
aliases: ["/docs/architecture/"]
weight: 1
---

Helm can be run in a secure, minimal container image based on scratch for maximum security. This approach provides an isolated environment with zero attack surface while maintaining full Helm functionality.

## Overview

The Helm container image offers several advantages over traditional installations:

- **Enhanced Security**: Minimal attack surface with scratch-based image
- **Isolation**: Complete separation from host system
- **Consistency**: Same Helm version across different environments
- **Zero Dependencies**: No need to install Helm directly on the host

## Quick Start

```bash
# Test Helm installation
docker run --rm helm:latest version --client

# Access local Kubernetes cluster
docker run --rm --net=host \
  -v ~/.kube/config:/tmp/kubeconfig:ro \
  -e KUBECONFIG=/tmp/kubeconfig \
  helm:latest list --all-namespaces
```

## Secure Container Configuration

Example of a security-hardened runtime configuration:

```bash
docker run --rm -it \
  --net=host \
  --read-only \
  --tmpfs /tmp:noexec,nosuid,nodev,size=50m \
  --security-opt=no-new-privileges \
  --cap-drop=ALL \
  --memory=256m \
  --cpus=1.0 \
  -v ~/.kube/config:/tmp/kubeconfig:ro \
  -v $(pwd):/workspace \
  -w /workspace \
  -e KUBECONFIG=/tmp/kubeconfig \
  helm:latest
```

## Usage Patterns

### Working with Local Files

When working with local charts or files, mount the necessary directories as volumes:

- **Current directory**: `-v $(pwd):/workspace -w /workspace`
- **Specific chart directory**: `-v /path/to/charts:/charts:ro`
- **Multiple directories**: Add multiple `-v` flags as needed

Use the secure configuration above with these volume mounts added.

### CI/CD Integration

For CI/CD pipelines, use the same secure configuration but remove the `-it` flags since interactive TTY is not needed in automated environments.

## Building the Container Image

Build the container using the no-context approach for maximum security:

```bash
# Build for current platform
cat Containerfile | docker build \
  --build-arg HELM_VERSION=v3.18.3 \
  -t helm:latest \
  -

# Multi-platform build
cat Containerfile | docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --build-arg HELM_VERSION=v3.18.3 \
  -t helm:latest \
  -
```

## Security Features

This container image provides enhanced security through:

- **Scratch base image**: Contains only the Helm binary and essential files
- **Non-root user**: Runs as UID 65532 for privilege separation  
- **Cryptographic verification**: PGP signatures and checksums verified during build
- **Multi-stage build**: Download tools excluded from final image
- **Zero attack surface**: No shell, libraries, or package managers included
- **Read-only filesystem**: Supports read-only root filesystem at runtime

## Shell Integration

Create a shell alias using the secure configuration above for convenient daily usage:

```bash
# Add to ~/.bashrc or ~/.zshrc
alias helm='docker run --rm -it \
  --net=host \
  --read-only \
  --tmpfs /tmp:noexec,nosuid,nodev,size=50m \
  --security-opt=no-new-privileges \
  --cap-drop=ALL \
  --memory=256m \
  --cpus=1.0 \
  -v ~/.kube/config:/tmp/kubeconfig:ro \
  -v $(pwd):/workspace \
  -w /workspace \
  -e KUBECONFIG=/tmp/kubeconfig \
  helm:latest'
```

After setting the alias, use Helm commands normally:
```bash
helm list
helm install my-release ./my-chart
```

## Troubleshooting

### Permission Issues

If you encounter permission issues with mounted volumes, replace the `--user=65532:65532` flag with your own user ID:

```bash
--user $(id -u):$(id -g)
```

### Network Connectivity

For different Kubernetes environments:

- **Local clusters** (kind, minikube, k3s): Use `--net=host` as shown in the secure configuration
- **Remote clusters**: Remove `--net=host` flag entirely
- **Custom networks**: Replace `--net=host` with `--network=my-kube-network`

### Offline Usage

For air-gapped environments:

```bash
# Save image to tar file
docker save helm:latest | gzip > helm-container.tar.gz

# Load on air-gapped system  
gunzip -c helm-container.tar.gz | docker load
```
