# Rocky9 Bootc Base Image

## Overview

This repository builds an **unofficial**  Rocky Linux 9 bootc base container image from scratch. Since Rocky does (at the time of writing) not currently provide an official bootc base image, this project assembles one using the Rocky RPM repositories and the bootc base image tooling.

The result is a bootc compatible container image that can later be used to generate an installable ISO, a disk image, or to install directly onto a system using bootc.

This document explains the build stages, what happens internally, and the role of the YAML manifest.

## How to use

We need to build this on a linux host or VM with a Linux kernel.

On a VM install a Fedora/Redhat/CentOS/Rocky with podman and then build the bootc case container.
```
podman build --platform="linux/amd64" --security-opt=label=disable --cap-add=all --device /dev/fuse --iidfile /tmp/image-id -t rocky-bootc -f 9/Containerfile 9
```

### Building AMD64 on Apple Silicon (local builds only)

On an Apple Silicon Mac the Podman machine has to emulate x86_64, and neither available emulator can run the whole build:

| Build step | Rosetta | QEMU |
|---|---|---|
| `rpm-ostree compose` (uses `bwrap`, needs Linux namespaces) | works | fails: `bwrap: Creating new namespace failed: Invalid argument` |
| `bootc container lint` | fails: `Linting: Function not implemented (os error 38)` | works |

The Rosetta failure started when the Podman machine image moved to a Linux kernel 6.13 or newer. The Podman machine image itself notes, in `/etc/systemd/system/rosetta-activation.service`, that Rosetta is not functional on those kernels. Related issue: https://github.com/podman-desktop/extension-bootc/issues/1475

The workaround is to build with Rosetta and skip the lint:

1. Make sure Rosetta is active in the Podman machine. Inside the VM, Rosetta is only activated when the marker file `/etc/containers/enable-rosetta` exists. The Podman Desktop "Rosetta" setting only applies when a machine is created, so for an existing machine create the marker and restart:
   ```
   export CONTAINERS_MACHINE_PROVIDER=applehv   # needed if containers.conf sets another provider, e.g. libkrun
   podman machine ssh podman-machine-default 'sudo touch /etc/containers/enable-rosetta'
   podman machine stop podman-machine-default
   podman machine start podman-machine-default
   podman machine ssh podman-machine-default 'ls /proc/sys/fs/binfmt_misc/'   # should list "rosetta"
   ```
2. Build with `SKIP_LINT=1`:
   ```
   podman build --platform="linux/amd64" --build-arg SKIP_LINT=1 --security-opt=label=disable --cap-add=all --device /dev/fuse -t rocky-bootc -f 9/Containerfile 9
   ```

`SKIP_LINT=1` removes the `bootc container lint` call from `bootc-base-imagectl build-rootfs` and skips the lint in the final stage. The default is `0`, so builds on a native x86_64 host still run the lint. Images built with `SKIP_LINT=1` are for local testing only. Build release images on a native x86_64 host.

## High Level Architecture

The build process consists of multiple container stages:

1. A stage that provides Rocky repository configuration.
2. A stage that provides the bootc base image build tooling.
3. A builder stage that assembles the target root filesystem.
4. A final scratch stage that contains only the assembled root filesystem.

The important concept is that we are constructing a complete operating system root filesystem inside a container image. That filesystem includes the kernel, systemd, and all required base packages. The container image itself is only a transport format for that filesystem.

## Stage 1: Rocky Repository Source

The first stage is based on a Rocky Linux 9 image.

Purpose:

- Provide official Rocky repository files.
- Provide the Rocky GPG keys.
- Ensure that all packages used during the build come from Rocky.

In this stage we do not build anything. We only extract:

- Files from /etc/yum.repos.d
- The Rocky GPG key from /etc/pki/rpm-gpg

These files are later copied into the builder stage so that dnf uses the correct Rocky repositories.

## Stage 2: bootc Base Image Tooling

The second stage is based on a CentOS bootc image that provides the helper tool:

bootc base imagectl

Purpose:

- Provide the tooling required to construct a bootable root filesystem.
- Provide reference manifests such as standard manifest.yaml.

From this stage we copy:

- The bootc base imagectl binary.
- The manifest templates.

This stage is only a tooling provider. It does not contribute packages to the final Rocky system.

## Stage 3: Builder Stage

The builder stage is where the actual Rocky bootc root filesystem is created.

Base image:

- Rocky Linux 9

In this stage we:

1. Install required build tools such as podman, bootc, ostree, and rpm ostree.
2. Remove any default repository configuration.
3. Copy in the Rocky repository files and GPG key from Stage 1.
4. Copy in bootc base imagectl and the manifest files from Stage 2.
5. Run the root filesystem build process.

The key command is:

bootc base imagectl build rootfs --manifest=rocky-9 /target-rootfs

This command performs the following steps:

- Reads the rocky-9 YAML manifest.
- Resolves all included manifests.
- Uses dnf to install the requested package set into /target-rootfs.
- Prepares the filesystem layout so it is bootc compatible.
- Ensures required boot integration components are present.

The directory /target-rootfs now contains a complete Rocky Linux operating system tree.

## Where the Kernel Comes From

The kernel is installed as a normal RPM during the build rootfs step.

Important points:

- The kernel is not taken from the host system.
- The kernel is not taken from the builder container runtime.
- The kernel is installed from the Rocky BaseOS repository via dnf.

During the root filesystem assembly:

- dnf installs kernel packages into /target-rootfs.
- Kernel modules are placed under /usr/lib/modules.
- Related components such as dracut and systemd are installed as required by the manifest.

When the resulting container image is later installed onto a machine, this kernel is copied to disk and configured by the bootloader.

When running the image as a container, the host kernel is used as usual. The kernel inside the image is only relevant when the image is installed as an operating system.

## Role of the YAML Manifest

The rocky-9.yaml file defines the operating system composition.

It specifies:

- Which repositories are enabled.
- Which packages are installed.
- Which additional manifests are included.

The most important line is:

include:
  standard/manifest.yaml

This means that rocky-9.yaml builds on top of a standard base definition. The standard manifest defines the core package set required for a bootable system. That typically includes:

- Kernel packages.
- systemd.
- Core system utilities.
- Boot related components.

The rocky-9.yaml acts as:

- A Rocky specific wrapper.
- A repository definition layer.
- A customization entry point.

In other words, the YAML manifest defines what the operating system contains. The bootc tooling only executes what the manifest describes.

Available manifests from the fedora project can be found here https://gitlab.com/fedora/bootc/base-images/-/blob/main/iot/manifest.yaml?ref_type=heads 

## Stage 4: Final Image

The final stage uses:

FROM scratch

We copy:

/target-rootfs

from the builder stage into the final image.

This means:

- The final image contains only the constructed operating system filesystem.
- No build tools are present.
- No extra layers from the builder remain.

The result is a clean bootc base container image that represents a full Rocky Linux 9 system.

## From Container to ISO or Disk Image

Once the bootc container image is built, it can be used to generate:

- An installable ISO.
- A qcow2 disk image.
- A raw disk image.

The ISO or disk image build must be performed on a machine that matches the target architecture.
a
During installation:

- The filesystem from the container image is written to disk.
- The kernel and modules installed during the rootfs build are deployed.
- The bootloader is configured to boot that kernel.

After installation, the system uses bootc for updates.

## Summary

This project constructs a Rocky Linux 9 bootc base image by:

1. Using official Rocky repositories.
2. Assembling a complete root filesystem using a YAML defined package set.
3. Embedding that filesystem into a minimal container image.
4. Allowing the image to be transformed into a bootable operating system.

The YAML manifest defines the content of the operating system. The bootc tooling executes the build. The kernel and all core components are installed as RPMs from the Rocky repositories during the root filesystem assembly.


## Good to know

How Alma Linux builds base bootc images 
https://github.com/AlmaLinux/bootc-images/blob/main/9/almalinux-9.yaml 