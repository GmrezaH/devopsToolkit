# Install Docker Engine

This guide provides step-by-step instructions for installing Docker Engine on Red Hat Enterprise Linux (RHEL) systems, focusing on containerization best practices for DevOps environments. It covers prerequisites, installation methods, and configurations to ensure efficient software delivery and infrastructure setup.

## Prerequisites

### OS requirements

To install Docker Engine, you need a maintained version of one of the following RHEL versions:

- RHEL 8
- RHEL 9

### Uninstall old versions

Before you can install Docker Engine, you need to uninstall any conflicting packages.

```bash
sudo dnf remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine \
                  podman \
                  runc
```

### Storage Configuration

Before installing Docker, create a separate partition for `/var/lib/docker` to optimize storage management and prevent issues with disk space in production environments. Refer to [linux/storage/README.md](../linux/storage/README.md) for detailed instructions on setting up this partition.

## Installation methods

You can install Docker Engine in different ways, depending on your needs:

- You can set up Docker's repositories and install from them, for ease of installation and upgrade tasks. This is the recommended approach.

- You can download the RPM package, install it manually, and manage upgrades completely manually. This is useful in situations such as installing Docker on air-gapped systems with no access to the internet.

### Install using the rpm repository

Before you install Docker Engine for the first time on a new host machine, you need to set up the Docker repository. Afterward, you can install and update Docker from the repository.

#### Set up the repository

Install the `dnf-plugins-core` package (which provides the commands to manage your DNF repositories) and set up the repository.

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

#### Install Docker Engine

1. Install the Docker packages

   ```bash
   sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

   If prompted to accept the GPG key, verify that the fingerprint matches `060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35`, and if so, accept it.

1. Start Docker Engine

   ```bash
   sudo systemctl enable --now docker
   ```

1. Verify that the installation is successful by running the `hello-world` image:

   ```bash
   sudo docker run hello-world
   ```

### Install from a package

If you can't use Docker's rpm repository to install Docker Engine, you can download the .rpm file for your release and install it manually. You need to download a new file each time you want to upgrade Docker Engine.

1. Go to https://download.docker.com/linux/rhel/

1. Select your RHEL version in the list.

1. Select the applicable architecture (x86_64, aarch64, or s390x), and then go to stable/Packages/.

1. Download the following `rpm` files for the Docker Engine, CLI, containerd, and Docker Compose packages:

   - `containerd.io-<version>.<arch>.rpm`
   - `docker-ce-<version>.<arch>.rpm`
   - `docker-ce-cli-<version>.<arch>.rpm`
   - `docker-buildx-plugin-<version>.<arch>.rpm`
   - `docker-compose-plugin-<version>.<arch>.rpm`

1. Install Docker Engine, changing the following path to the path where you downloaded the packages.

```bash
sudo dnf install ./containerd.io-<version>.<arch>.rpm \
  ./docker-ce-<version>.<arch>.rpm \
  ./docker-ce-cli-<version>.<arch>.rpm \
  ./docker-buildx-plugin-<version>.<arch>.rpm \
  ./docker-compose-plugin-<version>.<arch>.rpm
```

1. Start Docker Engine

   ```bash
   sudo systemctl enable --now docker
   ```

1. Verify that the installation is successful by running the `hello-world` image:

   ```bash
   sudo docker run hello-world
   ```

## Post-installation steps

These optional post-installation procedures describe how to configure your Linux host machine to work better with Docker.

### Manage Docker as a non-root user

The Docker daemon binds to a Unix socket, not a TCP port. By default it's the `root` user that owns the Unix socket, and other users can only access it using `sudo`. The Docker daemon always runs as the `root` user.

If you don't want to preface the `docker` command with `sudo`, create a Unix group called `docker` and add users to it. When the Docker daemon starts, it creates a Unix socket accessible by members of the `docker` group. On some Linux distributions, the system automatically creates this group when installing Docker Engine using a package manager. In that case, there is no need for you to manually create the group.

Deprecated features

Testcontainers
AI
Products
Testcontainers Cloud
Deprecated products and features
Release lifecycle
Platform
Release notes
Enterprise

Home
/
Manuals
/
Docker Engine
/
Install
/ Post-installation steps
Linux post-installation steps for Docker Engine
Page options

These optional post-installation procedures describe how to configure your Linux host machine to work better with Docker.
Manage Docker as a non-root user

The Docker daemon binds to a Unix socket, not a TCP port. By default it's the root user that owns the Unix socket, and other users can only access it using sudo. The Docker daemon always runs as the root user.

If you don't want to preface the docker command with sudo, create a Unix group called docker and add users to it. When the Docker daemon starts, it creates a Unix socket accessible by members of the docker group. On some Linux distributions, the system automatically creates this group when installing Docker Engine using a package manager. In that case, there is no need for you to manually create the group.

    Warning

    The docker group grants root-level privileges to the user. For details on how this impacts security in your system, see Docker Daemon Attack Surface.

    Note

    To run Docker without root privileges, see Run the Docker daemon as a non-root user (Rootless mode).

To create the `docker` group and add your user:

1. Create the `docker` group.

   ```bash
   sudo groupadd docker
   ```

1. Add your user to the `docker` group.

   ```bash
   sudo usermod -aG docker $USER
   ```

1. Log out and log back in so that your group membership is re-evaluated.

   ```
   If you're running Linux in a virtual machine, it may be necessary to restart the virtual machine for changes to take effect.
   ```

   You can also run the following command to activate the changes to groups:

   ```bash
   newgrp docker
   ```
