# Import a Docker Image into containerd

This guide shows how to export the `kube-apiserver` image from Docker and import it into containerd's `k8s.io` namespace, so Kubernetes can use it without pulling from a registry.

## Prerequisites

- Docker installed, with the image already available locally
- containerd installed, with the `ctr` CLI
- `sudo` access on the target node

## 1. Save the image from Docker

```bash
docker save -o kube.tar registry.k8s.io/kube-apiserver:v1.36.5
```

> **Note:** `-o kube.tar` sets the output file. The image name comes last.

## 2. Import the image into containerd

```bash
sudo ctr -n k8s.io images import kube.tar
```

The `-n k8s.io` flag selects the namespace Kubernetes uses. Without it, the image goes into the `default` namespace and the kubelet will not see it.

## 3. Verify the image

Check that containerd can see the image:

```bash
sudo ctr -n k8s.io images ls | grep kube-apiserver
```

You should see an image reference similar to:

```
registry.k8s.io/kube-apiserver:v1.36.5
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| No output from the `grep` | Confirm you used `-n k8s.io` on both the import and the list command |
| `ctr: command not found` | Install containerd, or use the full path (e.g. `/usr/bin/ctr`) |
| Permission denied | Run the `ctr` commands with `sudo` |
| Image missing in `docker save` | Check with `docker images` that the tag matches exactly |
