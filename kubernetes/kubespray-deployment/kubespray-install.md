# Installing Kubernetes with Kubespray

This guide, part of the [GmrezaH/devopsToolkit](https://github.com/GmrezaH/devopsToolkit) Kubernetes deployment documentation, provides step-by-step instructions for installing a Kubernetes cluster using Kubespray in both **air-gapped** and **online** environments. It covers cloning the Kubespray repository, Ansible setup, inventory configuration, variable adjustments (including options for Calico or Cilium CNI), and cluster deployment.

## Prerequisites

Before installing the cluster, ensure the following:

- An Ansible control machine prepared as described in [prepare-os.md](prepare-os.md).
- SSH keys generated and copied to all target nodes (see [prepare-os.md](prepare-os.md)).
- For air-gapped environments, a local Nexus repository with required files, images, and packages (see [air-gapped.md](air-gapped.md)).
- High-availability load balancer configured with a VIP for the Kubernetes API (see [loadbalancer.md](loadbalancer.md)).
- Define variables for node IPs (`${KUBE_NODE_1}`, `${KUBE_NODE_2}`, `${KUBE_NODE_3}`), Nexus IP (`${NEXUS_IP}`), VIP (`${vip}`), and other cluster-specific settings.

## Table of Contents

- [Configure local repository for air-gapped environments](#configure-local-repository-for-air-gapped-environments)
- [Set-up Kubespray environment](#set-up-kubespray-environment)
- [Generate SSH Key for Kubespray](#generate-ssh-key-for-kubespray)
- [Configure Kubespray Inventory](#configure-kubespray-inventory)
- [Configure Group Variables](#configure-group-variables)
- [Configure Hardening Variables](#configure-hardening-variables)
- [Example commands](#example-commands)
- [Deploy the Kubernetes Cluster](#deploy-the-kubernetes-cluster)
- [Verify Installation](#verify-installation)
- [Next Steps](#next-steps)

## Configure local repository for air-gapped environments

Configure Ansible host to use local repositories

- Configure pip to use the local Nexus PyPI repository:

  ```bash
  cat <<EOF > ~/.pip/pip.conf
  [global]
  index-url = http://${NEXUS_IP}:8081/repository/pypi/simple
  trusted-host = ${NEXUS_IP}
  EOF
  ```

- Configure Docker to use the local Nexus image registry:

  ```bash
  cat <<EOF > /etc/docker/daemon.json
  {
    "insecure-registries" : ["${NEXUS_IP}:8082"],
    "log-opts": {
      "max-size": "100M",
      "max-file": "5",
      "labels": "ansible-host"
      },
    "experimental": true,
    "live-restore": true
  }
  EOF
  ```

- Login to local Nexus image registry:

  ```bash
  sudo docker login ${NEXUS_IP}:8082
  ```

## Set-up Kubespray environment

On the Ansible control machine, set up the environment for Kubespray.

1. As Ansible is a python application, we will create a fresh virtual environment to install the dependencies for the Kubespray playbook:

   ```bash
   cd /opt/ansible
   VENVDIR=kubespray-venv
   KUBESPRAYDIR=kubespray
   python3.12 -m venv $VENVDIR
   source $VENVDIR/bin/activate
   ```

1. Clone the Kubespray code into current working directory (eg. /opt/ansible):

   - **Online environments**:

     Clone the repository and check out the supported version:

     ```bash
     git clone https://github.com/kubernetes-sigs/kubespray.git
     cd kubespray
     git checkout release-2.27
     ```

   - **Air-gapped environments**:

     Transfer the repository ZIP file to the Ansible control machine and extract and rename the repository. See [air-gapped.md](air-gapped.md) for additional details on transferring files in air-gapped environments.

     ```bash
     cd /opt/ansible
     unzip kubespray-release-2.27.1.zip
     mv kubespray-release-2.27.1 kubespray
     cd kubespray
     ```

1. Install the dependencies for Ansible to run the Kubespray playbook:

   ```bash
   pip3.12 install -r requirements.txt
   ```

## Generate SSH Key for Kubespray

To enhance security, generate a dedicated SSH key pair for Kubespray instead of using the VM's default key.

1. Generate a new ECDSA SSH key pair:

   ```bash
   ssh-keygen -t ecdsa -C "kubespray keys" -f ~/.ssh/kubespray_key -N ''
   ```

1. Copy the public key to all target nodes:

   ```bash
   ssh-copy-id -i ~/.ssh/kubespray_key.pub -p ${SSH_PORT} root@${KUBE_NODE_1}
   ssh-copy-id -i ~/.ssh/kubespray_key.pub -p ${SSH_PORT} root@${KUBE_NODE_2}
   ssh-copy-id -i ~/.ssh/kubespray_key.pub -p ${SSH_PORT} root@${KUBE_NODE_3}
   ```

> [!NOTE]
> Replace `${SSH_PORT}` with your SSH port (default: 22) and add more nodes as needed.

## Configure Kubespray Inventory

Create and configure the inventory for your cluster.

1. Copy the sample inventory:

   ```bash
   cp -rfp inventory/sample inventory/mycluster
   ```

2. Generate the inventory file:

   - Configure 'ip' variable to bind kubernetes services on a different ip than the default iface.
   - etcd_member_name should be set for etcd cluster. The node that are not etcd members do not need to set the value, or can set the empty string value.
   - See [Ansible Official docs](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html) for tips on building your inventory.

   ```bash
   cat <<EOF > inventory/mycluster/inventory.ini
   master1 ansible_host=95.54.0.12 ansible_port=1313 ip=10.3.0.1 etcd_member_name=etcd1
   master2 ansible_host=95.54.0.13 ansible_port=1313 ip=10.3.0.2 etcd_member_name=etcd2
   master3 ansible_host=95.54.0.14 ansible_port=1313 ip=10.3.0.3 etcd_member_name=etcd3
   worker1 ansible_host=95.54.0.15 ansible_port=1313 ip=10.3.0.4
   worker2 ansible_host=95.54.0.16 ansible_port=1313 ip=10.3.0.5
   worker3 ansible_host=95.54.0.17 ansible_port=1313 ip=10.3.0.6

   [kube_control_plane]
   master1
   master2
   master3

   [etcd]
   master1
   master2
   master3

   [kube_node]
   worker1
   worker2
   worker3
   EOF
   ```

## Configure Group Variables

Review and modify the group variables in `inventory/mycluster/group_vars` to suit your cluster.

1. Update `group_vars/all/all.yml`:

   ```yaml
   ---
   ## External LB config
   apiserver_loadbalancer_domain_name: "vip.example.com"
   loadbalancer_apiserver:
     address: ${vip}
     port: 6443

   ## Internal loadbalancers for apiservers
   loadbalancer_apiserver_localhost: true
   # valid options are "nginx" or "haproxy"
   loadbalancer_apiserver_type: nginx

   ## Local loadbalancer should use this port And must be set port 6443
   loadbalancer_apiserver_port: 6443

   ## If loadbalancer_apiserver_healthcheck_port variable defined, enables proxy liveness check for nginx.
   loadbalancer_apiserver_healthcheck_port: 8081

   ## Upstream dns servers
   upstream_dns_servers:
     - 8.8.8.8
     - 8.8.4.4

   ## Set these proxy values in order to update package manager and docker daemon to use proxies and custom CA for https_proxy if needed
   # http_proxy: ""
   # https_proxy: ""
   # https_proxy_cert_file: ""

   ## Refer to roles/kubespray-defaults/defaults/main/main.yml before modifying no_proxy
   # no_proxy: ""

   ## Set true to download and cache container
   download_container: true

   ## Deploy container engine
   # Set false if you want to deploy container engine manually.
   deploy_container_engine: true

   # sysctl_file_path to add sysctl conf to
   sysctl_file_path: "/etc/sysctl.d/99-sysctl.conf"

   ## NTP Settings
   # Start the ntpd or chrony service and enable it at system boot.
   ntp_enabled: true
   ntp_timezone: Asia/Tehran
   ntp_manage_config: true
   ntp_servers:
     - "0.pool.ntp.org iburst"
     - "1.pool.ntp.org iburst"
     - "2.pool.ntp.org iburst"
     - "3.pool.ntp.org iburst"
   ```

1. Update `group_vars/all/containerd.yml`:

   **NOTE:** Make sure to set both `IP` and `URL` for repositories.

   ```yaml
   ---
   # Registries defined within containerd.
   containerd_registries_mirrors:
     - prefix: nexus.example.com:8082
       mirrors:
         - host: http://nexus.example.com:8082
           capabilities: ["pull", "resolve"]
           skip_verify: false

   containerd_registry_auth:
     - registry: nexus.example.com:8082
       username: admin
       password: admin
   ```

1. Update `group_vars/all/etcd.yml`:

   ```yaml
   ---
   ## Directory where etcd data stored
   etcd_data_dir: /var/lib/etcd
   ## Container runtime
   ## docker for docker, crio for cri-o and containerd for containerd.
   ## Additionally you can set this to kubeadm if you want to install etcd using kubeadm
   ## Kubeadm etcd deployment is experimental and only available for new deployments
   ## If this is not set, container manager will be inherited from the Kubespray defaults
   ## and not from k8s_cluster/k8s-cluster.yml, which might not be what you want.
   ## Also this makes possible to use different container manager for etcd nodes.
   container_manager: containerd
   ## Settings for etcd deployment type
   # Set this to docker if you are using container_manager: docker
   # valid options are "host" or "kubeadm"
   etcd_deployment_type: kubeadm
   etcd_metrics_port: 2381
   etcd_listen_metrics_urls: "http://0.0.0.0:2381"
   etcd_metrics_service_labels:
     k8s-app: etcd
     app.kubernetes.io/managed-by: Kubespray
     app: kube-prometheus-stack-kube-etcd
     release: kube-prometheus-stack
   ```

1. Update `group_vars/all/offline.yml` for air-gapped environments:

   ```yaml
   ---
   registry_host: "nexus.example.com:8082"
   files_repo: "http://nexus.example.com:8081/repository/raw"

   ## Container Registry overrides
   kube_image_repo: "{{ registry_host }}/registry.k8s.io"
   gcr_image_repo: "{{ registry_host }}/gcr.io"
   github_image_repo: "{{ registry_host }}/ghcr.io"
   docker_image_repo: "{{ registry_host }}/docker.io"
   quay_image_repo: "{{ registry_host }}/quay.io"

   ## Kubernetes components
   kubeadm_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubeadm"
   kubectl_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubectl"
   kubelet_download_url: "{{ files_repo }}/dl.k8s.io/release/{{ kube_version }}/bin/linux/{{ image_arch }}/kubelet"

   ## Override entire binary repository
   github_url: "{{ files_repo }}/github.com"
   dl_k8s_io_url: "{{ files_repo }}/dl.k8s.io"
   storage_googleapis_url: "{{ files_repo }}/storage.googleapis.com"
   get_helm_url: "{{ files_repo }}/get.helm.sh"
   ```

1. Update `group_vars/k8s_cluster/addons.yml`:

   ```yaml
   ---
   # Kubernetes dashboard
   # RBAC required. see docs/getting-started.md for access details.
   dashboard_enabled: true

   # Helm deployment
   helm_enabled: true

   # Metrics Server deployment
   metrics_server_enabled: true
   metrics_server_container_port: 10250
   metrics_server_kubelet_insecure_tls: true
   metrics_server_metric_resolution: 15s
   metrics_server_kubelet_preferred_address_types: "InternalIP,ExternalIP,Hostname"
   metrics_server_host_network: false
   metrics_server_replicas: 1
   metrics_server_extra_affinity:
     nodeAffinity:
       requiredDuringSchedulingIgnoredDuringExecution:
         nodeSelectorTerms:
           - matchExpressions:
               - key: node-role.kubernetes.io/control-plane
                 operator: Exists
   ```

   > **NOTE:**
   > The `metrics_server_extra_affinity` setting ensures Metrics Server pods run only on control plane nodes, as `kubelet_secure_addresses` in hardening.yml includes **only** control plane IPs.
   >
   > If Metrics Server is scheduled on a worker node, it can only scrape metrics from that specific worker and its pods, and fail to scrape other nodes and pods.

1. Update `group_vars/k8s_cluster/k8s-cluster.yml` with CNI options (choose Calico or Cilium):

   ```yaml
   ---
   # This is the user that owns tha cluster installation.
   kube_owner: root

   # Choose network plugin (cilium, calico, kube-ovn, weave or flannel. Use cni for generic cni plugin)
   # Can also be set to 'cloud', which lets the cloud provider setup appropriate routing
   kube_network_plugin: cilium

   # Kubernetes internal network for services, unused block of space.
   kube_service_addresses: 10.232.0.0/18

   # internal network. When used, it will assign IP
   # addresses from this range to individual pods.
   # This network must be unused in your network infrastructure!
   kube_pods_subnet: 10.233.0.0/16

   # internal network node size allocation (optional). This is the size allocated
   # to each node for pod IP address allocation. Note that the number of pods per node is
   # also limited by the kubelet_max_pods variable which defaults to 110.
   #
   # Example:
   # Up to 64 nodes and up to 254 or kubelet_max_pods (the lowest of the two) pods per node:
   #  - kube_pods_subnet: 10.233.64.0/18
   #  - kube_network_node_prefix: 24
   #  - kubelet_max_pods: 110
   #
   # Example:
   # Up to 128 nodes and up to 126 or kubelet_max_pods (the lowest of the two) pods per node:
   #  - kube_pods_subnet: 10.233.64.0/18
   #  - kube_network_node_prefix: 25
   #  - kubelet_max_pods: 110
   kube_network_node_prefix: 24

   # Kube-proxy proxyMode configuration.
   # Can be ipvs, iptables
   kube_proxy_mode: ipvs

   # configure arp_ignore and arp_announce to avoid answering ARP queries from kube-ipvs0 interface
   # must be set to true for MetalLB, kube-vip(ARP enabled) to work
   kube_proxy_strict_arp: true

   # Graceful Node Shutdown (Kubernetes >= 1.21.0), see https://kubernetes.io/blog/2021/04/21/graceful-node-shutdown-beta/
   # kubelet_shutdown_grace_period had to be greater than kubelet_shutdown_grace_period_critical_pods to allow
   # non-critical podsa to also terminate gracefully
   kubelet_shutdown_grace_period: 60s
   kubelet_shutdown_grace_period_critical_pods: 20s

   # DNS configuration.
   # Kubernetes cluster name, also will be used as DNS domain
   cluster_name: cluster.local

   ## Container runtime
   ## docker for docker, crio for cri-o and containerd for containerd.
   ## Default: containerd
   container_manager: containerd

   # K8s image pull policy (imagePullPolicy)
   k8s_image_pull_policy: IfNotPresent

   # audit log for kubernetes
   kubernetes_audit: true

   # Make a copy of kubeconfig on the host that runs Ansible in {{ inventory_dir }}/artifacts
   # kubeconfig_localhost: false
   # Use ansible_host as external api ip when copying over kubeconfig.
   # kubeconfig_localhost_ansible_host: false
   # Download kubectl onto the host that runs Ansible in {{ bin_dir }}
   # kubectl_localhost: false

   # Whether to run kubelet and container-engine daemons in a dedicated cgroup.
   kube_reserved: true
   ## Uncomment to override default values
   ## The following two items need to be set when kube_reserved is true
   kube_reserved_cgroups_for_service_slice: kube.slice
   kube_reserved_cgroups: "/{{ kube_reserved_cgroups_for_service_slice }}"
   kube_memory_reserved: 256Mi
   kube_cpu_reserved: 100m
   kube_ephemeral_storage_reserved: 2Gi
   kube_pid_reserved: "1000"
   # Reservation for master hosts
   kube_master_memory_reserved: 512Mi
   kube_master_cpu_reserved: 200m
   kube_master_ephemeral_storage_reserved: 2Gi
   kube_master_pid_reserved: "1000"

   ## Optionally reserve resources for OS system daemons.
   system_reserved: true
   ## Uncomment to override default values
   ## The following two items need to be set when system_reserved is true
   system_reserved_cgroups_for_service_slice: system.slice
   system_reserved_cgroups: "/{{ system_reserved_cgroups_for_service_slice }}"
   system_memory_reserved: 512Mi
   system_cpu_reserved: 500m
   system_ephemeral_storage_reserved: 2Gi
   ## Reservation for master hosts
   system_master_memory_reserved: 256Mi
   system_master_cpu_reserved: 250m
   system_master_ephemeral_storage_reserved: 2Gi

   ## Eviction Thresholds to avoid system OOMs
   # https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#eviction-thresholds
   # eviction_hard: {}
   # eviction_hard_control_plane: {}

   ## Supplementary addresses that can be added in kubernetes ssl keys.
   ## That can be useful for example to setup a keepalived virtual IP
   # supplementary_addresses_in_ssl_keys:
   #   - ${VIP_IP}
   #   - ${MASTER1_IP}
   #   - ${MASTER2_IP}
   #   - ${MASTER3_IP}
   #   - ${VIP_FQDN}
   #   - ${VIP_HOSTNAME}
   #   - ${MASTER1_FQDN}
   #   - ${MASTER2_FQDN}
   #   - ${MASTER3_FQDN}
   #   - ${MASTER1_HOSTNAME}
   #   - ${MASTER2_HOSTNAME}
   #   - ${MASTER3_HOSTNAME}
   supplementary_addresses_in_ssl_keys:
     - 95.54.0.100
     - 95.54.0.12
     - 95.54.0.13
     - 95.54.0.14
     - vip.example.com
     - vip.example
     - vip
     - master1
     - master2
     - master3

   ## Support tls min version, Possible values: VersionTLS10, VersionTLS11, VersionTLS12, VersionTLS13.
   tls_min_version: "VersionTLS12"

   ## Amount of time to retain events. (default 1h0m0s)
   event_ttl_duration: "1h0m0s"

   ## Automatically renew K8S control plane certificates on first Monday of each month
   auto_renew_certificates: true
   # First Monday of each month
   auto_renew_certificates_systemd_calendar: "Mon *-*-1,2,3,4,5,6,7 03:{{ groups['kube_control_plane'].index(inventory_hostname) }}0:00"

   # System upgrade configuration
   system_upgrade: true
   system_upgrade_reboot: never
   ```

   > **Note**: Select Calico for simplicity or Cilium for advanced features like eBPF-based networking. Update `kube_network_plugin` accordingly.

1. Update `group_vars/k8s_cluster/k8s-net-calico.yml` for Calico CNI:

   ```yaml
   ---
   # If you want to use non default IP_AUTODETECTION_METHOD, IP6_AUTODETECTION_METHOD for calico node set this option to one of:
   # * can-reach=DESTINATION
   # * interface=INTERFACE-REGEX
   # see https://docs.projectcalico.org/reference/node/configuration
   calico_ip_auto_method: "interface=ens.*"
   calico_ip6_auto_method: "interface=ens.*"

   # Choose the iptables insert mode for Calico: "Insert" or "Append".
   calico_felix_chaininsertmode: Insert

   # Enable calico traffic encryption with wireguard
   calico_wireguard_enabled: false

   # Under certain situations liveness and readiness probes may need tunning
   calico_node_livenessprobe_timeout: 10
   calico_node_readinessprobe_timeout: 30
   ```

1. Update `group_vars/k8s_cluster/k8s-net-cilium.yml` for **Cilium CNI**:

   ```yaml
   # Kube Proxy Replacement mode (strict/partial)
   cilium_kube_proxy_replacement: true

   # Unique ID of the cluster. Must be unique across all connected clusters and
   # in the range of 1 and 255. Only relevant when building a mesh of clusters.
   # This value is not defined by default
   # cilium_cluster_id:

   # Hubble
   ### Enable Hubble without install
   cilium_enable_hubble: true
   ### Enable Hubble Metrics
   cilium_enable_hubble_metrics: true
   ### if cilium_enable_hubble_metrics: true
   # cilium_hubble_metrics: {}
   # - dns
   # - drop
   # - tcp
   # - flow
   # - icmp
   # - http
   ### Enable Hubble install
   cilium_hubble_install: true
   ### Enable auto generate certs if cilium_hubble_install: true
   cilium_hubble_tls_generate: true

   cilium_operator_replicas: 2
   # Name of the cluster. Only relevant when building a mesh of clusters.
   # cilium_cluster_name: default

   # Configure how long to wait for the Cilium DaemonSet to be ready again
   ###READ docs/cilium.md !!!!###
   cilium_rolling_restart_wait_retries_count: 30
   cilium_rolling_restart_wait_retries_delay_seconds: 10
   ```

1. Update `roles/kubespray_defaults/defaults/main/download.yml`:

   ```yaml
   ---
   download_keep_remote_cache: true
   skip_kubeadm_images: true
   download_run_once: true
   download_localhost: true
   image_command_tool_on_localhost: docker
   ```

1. Update `roles/kubespray_defaults/defaults/main/main.yml`:

   ```yaml
   ---
   container_manager_on_localhost: docker
   ```

## Configure Hardening Variables

Create a hardening configuration file `inventory/mycluster/hardening.yml` for security best practices.

> [!NOTE]
> Make sure to add kubernetes control-plane nodes IPs to kubelet_secure_addresses.

```yaml
---
## kube-apiserver
authorization_modes: ["Node", "RBAC"]
# AppArmor-based OS
# kube_apiserver_feature_gates: ['AppArmor=true']
kube_apiserver_request_timeout: 120s
kube_apiserver_service_account_lookup: true

# enable kubernetes audit
kubernetes_audit: true
audit_log_path: "/var/log/kube-apiserver-log.json"
audit_log_maxage: 30
audit_log_maxbackups: 10
audit_log_maxsize: 500

tls_min_version: VersionTLS12
tls_cipher_suites:
  - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305

# enable encryption at rest
kube_encrypt_secret_data: true
kube_encryption_resources: [secrets]
kube_encryption_algorithm: "secretbox"

kube_apiserver_enable_admission_plugins:
  - EventRateLimit
  - AlwaysPullImages
  - ServiceAccount
  - NamespaceLifecycle
  - NodeRestriction
  - LimitRanger
  - ResourceQuota
  - MutatingAdmissionWebhook
  - ValidatingAdmissionWebhook
  - PodNodeSelector
  - PodSecurity
kube_apiserver_admission_control_config_file: true
# Creates config file for PodNodeSelector
# kube_apiserver_admission_plugins_needs_configuration: [PodNodeSelector]
# Define the default node selector, by default all the workloads will be scheduled on nodes
# with label network=srv1
# kube_apiserver_admission_plugins_podnodeselector_default_node_selector: "network=srv1"
# EventRateLimit plugin configuration
kube_apiserver_admission_event_rate_limits:
  limit_1:
    type: Namespace
    qps: 50
    burst: 100
    cache_size: 2000
  limit_2:
    type: User
    qps: 50
    burst: 100
kube_profiling: false
# Remove anonymous access to cluster
# remove_anonymous_access: true

## kube-controller-manager
kube_controller_manager_bind_address: 0.0.0.0
kube_controller_terminated_pod_gc_threshold: 50
# AppArmor-based OS
# kube_controller_feature_gates: ["RotateKubeletServerCertificate=true", "AppArmor=true"]
kube_controller_feature_gates: ["RotateKubeletServerCertificate=true"]

## kube-scheduler
kube_scheduler_bind_address: 0.0.0.0
# AppArmor-based OS
# kube_scheduler_feature_gates: ["AppArmor=true"]

## etcd
# etcd_deployment_type: kubeadm

## kubelet
kubelet_authorization_mode_webhook: true
kubelet_authentication_token_webhook: true
kube_read_only_port: 0
# kubelet_rotate_server_certificates: true
kubelet_protect_kernel_defaults: true
kubelet_event_record_qps: 1
# kubelet_rotate_certificates: true
kubelet_streaming_connection_idle_timeout: "5m"
kubelet_make_iptables_util_chains: true
kubelet_feature_gates: ["RotateKubeletServerCertificate=true"]
kubelet_seccomp_default: true
kubelet_systemd_hardening: true
# In case you have multiple interfaces in your
# control plane nodes and you want to specify the right
# IP addresses, kubelet_secure_addresses allows you
# to specify the IP from which the kubelet
# will receive the packets.
kubelet_secure_addresses: "localhost link-local {{ kube_pods_subnet }} ${KUBE_CONTROLPLANE_IPS}"

# additional configurations
kube_owner: root
kube_cert_group: root

# create a default Pod Security Configuration and deny running of insecure pods
# kube_system namespace is exempted by default
kube_pod_security_use_default: true
kube_pod_security_default_enforce: restricted
```

## Example commands

[Kubespray Ansible tags](https://kubespray.io/#/docs/ansible/ansible?id=ansible-tags): The following tags are defined in playbooks.

- Example command to filter and apply only DNS configuration tasks and skip everything else related to host OS configuration and downloading images of containers:

  ```bash
  ansible-playbook -i inventory/mycluster/inventory.ini --tags preinstall,facts --skip-tags=download,bootstrap-os
  ```

- And this play only removes the K8s cluster DNS resolver IP from hosts' /etc/resolv.conf files:

  ```bash
  ansible-playbook -i inventory/mycluster/inventory.ini -e dns_mode='none' cluster.yml --tags resolvconf
  ```

- And this prepares all container images locally (at the ansible runner node) without installing or upgrading related stuff or trying to upload container to K8s cluster nodes:

  ```bash
  ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
      -e download_run_once=true -e download_localhost=true \
      --tags download --skip-tags upload,upgrade
  ```

## Deploy the Kubernetes Cluster

Run the Ansible playbooks to deploy the cluster.

1. Fix module link if needed.

   If the the `library/kube.py` is not a soft link, or if you faced the this issue: `module (kube) is missing interpreter line`, fix it with below instruction:

   ```bash
   cd library
   rm -f kube.py
   ln -s ../plugins/modules/kube.py .
   ```

1. Reset any existing cluster on the nodes (if applicable):

   ```bash
   ansible-playbook -v reset.yml \
           -i inventory/mycluster/inventory.ini \
           -b --become-user=root \
           --private-key ~/.ssh/kubespray_key
   ```

1. Prepare container images locally (optional, for air-gapped):

   ```bash
   ansible-playbook -v cluster.yml \
           -i inventory/mycluster/inventory.ini \
           -b --become-user=root \
           --private-key ~/.ssh/kubespray_key \
           -e download_run_once=true \
           -e download_localhost=true \
           --tags download \
           --skip-tags upload,upgrade
   ```

1. Deploy the cluster:

   ```bash
   ansible-playbook -v cluster.yml \
           -i inventory/mycluster/inventory.ini \
           -b --become-user=root \
           --private-key ~/.ssh/kubespray_key \
           -e "@hardening.yml"
   ```

## Verify Installation

1. Check cluster status:

   ```bash
   kubectl cluster-info
   ```

1. Check cluster nodes:

   ```bash
   kubectl get nodes
   ```

1. Verify pods:

   ```bash
   kubectl get pods --all-namespaces
   ```

1. Run security checks with kube-bench (post-deployment):

   ```bash
   kube-bench run --targets master,worker
   ```

## Next Steps

- Install add-ons like ingress-nginx, cert-manager (see [addons](../addons/README.md)).
- For cluster maintenance, see [kubespray-maintenance.md](kubespray-maintenance.md).
- Verify all steps with [checklist.md](checklist.md).

---

**Keywords**: Kubernetes installation, Kubespray deployment, Kubespray repository cloning, Ansible Kubernetes setup, Calico CNI, Cilium CNI, modular CNI configuration, SSH key management, air-gapped Kubernetes, hardening Kubernetes, DevOps, RHEL, Rocky Linux
