# Replicating a Podman Pod to Another Machine

This guide explains how to replicate (migrate/copy) an existing Podman pod from your Bazzite laptop to another Linux machine.

## Prerequisites

*   **Source Machine:** Bazzite (running Podman)
*   **Target Machine:** Linux with Podman installed
*   **Network Access:** SSH access between machines is recommended for file transfer.

## Step 1: Identify Your Pod

List your current pods to find the name of the one you want to replicate.

```bash
podman pod ps
```

*In this guide, we will use `my-pod` as the example name.*

## Step 2: Export Pod Configuration (YAML)

Podman can generate a Kubernetes-compatible YAML file that describes your pod and its containers.

```bash
podman generate kube my-pod > my-pod.yaml
```

*Note: If you want to include credential secrets or other specific configurations, check `podman generate kube --help` for more options.*

## Step 3: Save Container Images

The target machine needs the images used by your pod. You can save them to a tar archive.

First, list images to see which ones are used:
```bash
podman image ls
```

Then save the relevant images:
```bash
# Syntax: podman save -o <output-file.tar> <image1> <image2> ...
podman save -o my-pod-images.tar registry.fedoraproject.org/image1:latest docker.io/library/image2:tag
```

## Step 4: Backup Persistent Data (Volumes)

If your pod uses matching volumes or bind mounts, you must copy current data manually.

### Check for Volumes
Inspect the YAML file generated in Step 2 (`my-pod.yaml`) or inspect the pod containers to see where data is stored.

```bash
podman pod inspect my-pod
```

### Option A: Bind Mounts (Host Directories)
If you mapped a host directory (e.g., `-v /home/user/data:/data`), simply copy that directory using `scp` or `rsync` (see Step 5).

### Option B: Named Volumes
If you used named volumes, you need to export their data.
```bash
# Example: Mount the volume to a temporary container and tar the contents
podman run --rm -v my-volume-name:/volume -v $(pwd):/backup alpine tar cvf /backup/my-volume-data.tar -C /volume .
```

## Step 5: Transfer Files to Target Machine

Use `scp` or `rsync` to move the files to the new machine.

```bash
# Replace 'user@target-machine-ip' with your actual target details
scp my-pod.yaml my-pod-images.tar my-volume-data.tar user@target-machine-ip:~/
```

## Step 6: Restore on Target Machine

On the **target machine**, perform the following steps:

### 1. Load Images
```bash
podman load -i my-pod-images.tar
```

### 2. Restore Volumes
**For Bind Mounts:**  
Ensure the directory structure exists at the same path as the source, or edit `my-pod.yaml` to point to the new location.

**For Named Volumes:**  
Create the volume and restore data:
```bash
podman volume create my-volume-name
podman run --rm -v my-volume-name:/volume -v $(pwd):/backup alpine tar xvf /backup/my-volume-data.tar -C /volume
```

### 3. Deploy the Pod
Use the YAML file to recreate the pod.

```bash
podman play kube my-pod.yaml
```

## Verification

Check if the pod is running correctly on the new machine:

```bash
podman pod ps
podman pod logs my-pod
```
