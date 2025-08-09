# Preparing Operating System for Kubernetes with Kubespray

This guide, part of the [GmrezaH/devopsToolkit](https://github.com/GmrezaH/devopsToolkit) Kubernetes deployment documentation, outlines the steps to prepare nodes for a Kubernetes cluster using Kubespray in both **air-gapped** and **online** environments. It covers configuring the operating system, checking DNS, NTP, and OS versions, and setting up prerequisites for Kubespray deployment.

## Prerequisites

Before preparing nodes, ensure the following:

- All nodes (control plane, worker, and load balancer) run a supported Linux distribution, (This document uses RHEL/Rocky 9.6). See [Kubespray Kernel Requirements](https://kubespray.io/#/docs/operations/kernel-requirements).
- Root or sudo access to all nodes.
- For air-gapped environments, a local Nexus repository with required RPMs and packages (see [air-gapped.md](air-gapped.md)).
- Storage configuration completed, as described in [../../linux/storage/README.md](../../linux/storage/README.md).

## Table of Contents

- [Disable SELinux](#disable-selinux)
- [Update System Packages](#update-system-packages)
- [Check DNS Configuration](#check-dns-configuration)
- [Check NTP Configuration](#check-ntp-configuration)
- [Check OS Version](#check-os-version)
- [Configure Firewall Rules](#configure-firewall-rules)
- [Set Up Storage](#set-up-storage)
- [Next Steps](#next-steps)

## Disable SELinux

Kubespray requires SELinux to be disabled or set to permissive mode to avoid conflicts with Kubernetes components.

1. Check the current SELinux status:

   ```bash
   getenforce
   ```

1. Set SELinux to permissive mode temporarily:

   ```bash
   sudo setenforce 0
   ```

1. Update the SELinux configuration for persistence:

   ```bash
   sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
   ```

1. Verify the change:

   ```bash
   grep ^SELINUX= /etc/selinux/config
   ```

## Update System Packages

Ensure all nodes have the latest packages to avoid compatibility issues.

1. Update the package cache and upgrade packages:

   - **Online environments**:

     ```bash
     sudo dnf update -y
     ```

   - **Air-gapped environments**: Configure the local Nexus repository as described in [air-gapped.md](air-gapped.md), then run:

     ```bash
     sudo dnf --disablerepo="*" --enablerepo="nexus-local" update -y
     ```

1. Reboot if kernel updates were applied:

   ```bash
   sudo reboot
   ```

## Check DNS Configuration

Verify that DNS resolution is correctly configured on all nodes to ensure Kubernetes services can communicate.

1. Check the current DNS servers:

   ```bash
   resolvectl status | grep "DNS Servers"
   ```

   Example output:

   ```
   DNS Servers: 192.168.1.1
   ```

1. If DNS servers are incorrect or missing, update `/etc/resolv.conf`:

   - For online environments, use your network’s DNS server (e.g., `8.8.8.8` for Google DNS).
   - For air-gapped environments, use the local DNS server.

   ```bash
   sudo bash -c 'echo "nameserver 192.168.1.1" > /etc/resolv.conf'
   ```

1. Test DNS resolution:

   ```bash
   nslookup kubernetes.default
   ```

## Check NTP Configuration

Ensure time synchronization is enabled to prevent clock drift, which can cause issues with Kubernetes components.

1. Verify NTP service status:

   ```bash
   timedatectl
   ```

   Example output:

   ```
   Local time: Wed 2025-08-06 13:26:00 +04
   Universal time: Wed 2025-08-06 09:26:00 UTC
   RTC time: Wed 2025-08-06 09:26:00
   Time zone: Asia/Dubai (+04, +0400)
   System clock synchronized: yes
   NTP service: active
   RTC in local TZ: no
   ```

1. If NTP is not active, enable and configure it:

   - Install `chrony` if not present:

     ```bash
     sudo dnf install -y chrony
     ```

     - For online environments, use default NTP servers or your network’s servers.
     - For air-gapped environments, configure a local NTP server.

       ```bash
       sudo bash -c 'echo "server ntp.local iburst" > /etc/chrony.conf'
       sudo systemctl enable --now chronyd
       ```

1. Verify synchronization:

   ```bash
   timedatectl | grep "System clock synchronized"
   ```

## Check OS Version

Confirm that all nodes are running a supported OS version (This document uses RHEL/Rocky 9.6). See [Kubespray Kernel Requirements](https://kubespray.io/#/docs/operations/kernel-requirements).

1. Check the OS version:

   ```bash
   cat /etc/os-release
   ```

   Example output:

   ```
   NAME="Rocky Linux"
   VERSION="9.6 (Blue Onyx)"
   ID="rocky"
   ID_LIKE="rhel centos fedora"
   VERSION_ID="9.6"
   ```

## Configure Firewall Rules

Configure firewall rules to allow Kubernetes communication.

1. Install `firewalld` if not present

   ```bash
   sudo dnf install -y firewalld
   sudo systemctl enable --now firewalld
   ```

1. Choose a Suitable Zone

   It's common to use the `trusted` zone for fully trusted sources. However, if you prefer a different zone (e.g., `internal`), ensure it's configured appropriately. For this example, we'll use the `trusted` zone.

1. Add Trusted IPs and Networks to the Zone

   Add trusted sources to the trusted zone on all nodes for Kubernetes nodes, load balancer, and internal networks:

   ```bash
   # Add the trusted Load Balancer and Kubernetes nodes
   firewall-cmd --zone=trusted --add-source=192.168.200.10/32 --permanent
   firewall-cmd --zone=trusted --add-source=192.168.200.11/32 --permanent
   firewall-cmd --zone=trusted --add-source=192.168.200.12/32 --permanent
   firewall-cmd --zone=trusted --add-source=192.168.200.13/32 --permanent

   # Add Kubernetes internal networks for services and pods
   firewall-cmd --zone=trusted --add-source=10.232.0.0/18 --permanent
   firewall-cmd --zone=trusted --add-source=10.233.0.0/16 --permanent
   ```

   > **NOTE:**
   > Replace the example IPs (e.g., 192.168.200.10) with your actual Load Balancer and Kubernetes node IPs. Use your cluster's pod and service CIDRs for internal networks (see kubespray-install.md). If your cluster has many nodes and frequent additions/removals, add the Kubernetes network subnet to the trusted zone to reduce manual work, e.g.:
   >
   > ```bash
   > firewall-cmd --zone=trusted --add-source=192.168.200.0/24 --permanent
   > ```

1. Reload firewalld to apply changes

   ```bash
   firewall-cmd --reload
   ```

1. Verify the configuration

   - Active zones and interfaces

     ```bash
     firewall-cmd --get-active-zones
     ```

     Ensure the interface for the VIP (e.g., ens192) is in the public zone on LB VMs.

   - Trusted sources in the `trusted` zone

     ```bash
     firewall-cmd --zone=trusted --list-sources
     ```

## Set Up Storage

Kubernetes requires sufficient storage for logs, container images, and other data. To create a separate partition for `/var` or extend the existing one, follow the instructions in [Linux storage docs](../../linux/storage/README.md).

## Next Steps

- Configure the high-availability load balancer in [loadbalancer.md](loadbalancer.md).
- Deploy the cluster in [kubespray-install.md](kubespray-install.md).
- Use [kubespray-maintenance.md](kubespray-maintenance.md) for ongoing cluster management.
- Verify all steps with [checklist.md](checklist.md).

---

**Keywords**: Kubernetes node preparation, Kubespray setup, DNS configuration, NTP configuration, OS version check, air-gapped, online deployment, DevOps, Ansible, RHEL, Rocky Linux
