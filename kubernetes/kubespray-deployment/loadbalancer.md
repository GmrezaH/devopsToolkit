# Configuring High-Availability Load Balancer for Kubernetes with HAProxy and Keepalived

This guide, part of the [GmrezaH/devopsToolkit](https://github.com/GmrezaH/devopsToolkit) Kubernetes deployment documentation, details how to set up HAProxy and Keepalived for a high-availability load balancer serving the Kubernetes API and ingress traffic in both **air-gapped** and **online** environments. It includes firewall configuration and traffic forwarding for ingress load balancing to Kubernetes nodes.

## Prerequisites

Before configuring the load balancer, ensure the following:

- Two VMs dedicated for HAProxy and Keepalived, prepared as described in [prepare-os.md](prepare-os.md).
- A free virtual IP (VIP) in the same subnet as the Kubernetes nodes (e.g., `${vip}`).
- For air-gapped environments, access to the local Nexus repository for packages (see [air-gapped.md](air-gapped.md)).
- Define variables for master node IPs (`${master1_ip}`, `${master2_ip}`, `${master3_ip}`), worker node IPs (e.g., `${worker1_ip}`), VIP (`${vip}`), and network interface (e.g., `ens192`) based on your cluster setup.

## Table of Contents

- [Install HAProxy and Keepalived](#install-haproxy-and-keepalived)
- [Configure Firewall Rules](#configure-firewall-rules)
- [Configure HAProxy](#configure-haproxy)
- [Configure Keepalived on Master](#configure-keepalived-on-master)
- [Configure Keepalived on Backup](#configure-keepalived-on-backup)
- [Verify Configuration](#verify-configuration)
- [Next Steps](#next-steps)

## Install HAProxy and Keepalived

On both load balancer VMs, install the required packages.

1. Install HAProxy and Keepalived:

   - **Online environments**:

     ```bash
     sudo dnf install -y haproxy keepalived
     ```

   - **Air-gapped environments**: Use the local Nexus repository (see [air-gapped.md](air-gapped.md)):

     ```bash
     sudo dnf --disablerepo="*" --enablerepo="nexus-local" install -y haproxy keepalived
     ```

## Configure Firewall Rules

Configure `firewalld` on both load balancer VMs to allow HAProxy and Keepalived traffic, including Kubernetes API and ingress ports, aligning with [prepare-os.md](prepare-os.md).

1. Install `firewalld` if not present:

   ```bash
   sudo dnf install -y firewalld
   sudo systemctl enable --now firewalld
   ```

1. Open ports 80 (HTTP), 443 (HTTPS), 8000 (HAProxy stats), and 6443 (Kubernetes API) in the `public` zone:

   ```bash
   firewall-cmd --zone=public --add-port={80/tcp,443/tcp,8000/tcp,6443/tcp} --permanent
   ```

1. Reload the firewall:

   ```bash
   firewall-cmd --reload
   ```

1. Verify the configuration:

   - Trusted sources in the `trusted` zone:

     ```bash
     firewall-cmd --zone=trusted --list-sources
     ```

   - Open ports in the `public` zone:

     ```bash
     firewall-cmd --zone=public --list-ports
     ```

   - Active zones and interfaces:

     ```bash
     firewall-cmd --get-active-zones
     ```

     Ensure the network interface (e.g., `ens192`) is in the `public` zone for ports 80, 443, 8000, and 6443.

   - Enabled services in the public zone (ensure ssh is listed):

     ```bash
     firewall-cmd --zone=public --list-services
     ```

     Expected output includes `ssh`

## Configure HAProxy

On both load balancer VMs, configure HAProxy to handle Kubernetes API and ingress traffic, forwarding port 80 to 30080 and port 443 to 30081 on Kubernetes nodes.

1. Append the configuration to `/etc/haproxy/haproxy.cfg`:

   ```bash
   cat <<EOF >> /etc/haproxy/haproxy.cfg
   listen Stats-Page
     bind *:8000
     mode http
     stats enable
     stats hide-version
     stats refresh 10s
     stats uri /
     stats show-legends
     stats show-node
     stats admin if LOCALHOST
     stats auth admin:admin

   frontend fe-apiserver
     bind 0.0.0.0:6443
     mode tcp
     option tcplog
     default_backend be-apiserver

   backend be-apiserver
     mode tcp
     option tcp-check
     balance roundrobin
     default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
     server master1 ${master1_ip}:6443 check
     server master2 ${master2_ip}:6443 check
     server master3 ${master3_ip}:6443 check

   frontend fe-ingress-http
     bind 0.0.0.0:80
     mode tcp
     option tcplog
     default_backend be-ingress-http

   backend be-ingress-http
     mode tcp
     option tcp-check
     balance roundrobin
     default-server inter 10s downinter 5s rise 2 fall 2 maxconn 250 maxqueue 256 weight 100
     server master1 ${master1_ip}:30080 check
     server master2 ${master2_ip}:30080 check
     server master3 ${master3_ip}:30080 check
     server worker1 ${worker1_ip}:30080 check

   frontend fe-ingress-https
     bind 0.0.0.0:443
     mode tcp
     option tcplog
     default_backend be-ingress-https

   backend be-ingress-https
     mode tcp
     option tcp-check
     balance roundrobin
     default-server inter 10s downinter 5s rise 2 fall 2 maxconn 250 maxqueue 256 weight 100
     server master1 ${master1_ip}:30081 check
     server master2 ${master2_ip}:30081 check
     server master3 ${master3_ip}:30081 check
     server worker1 ${worker1_ip}:30081 check
   EOF
   ```

1. Verify the configuration:

   ```bash
   sudo haproxy -c -f /etc/haproxy/haproxy.cfg
   ```

1. Enable and start HAProxy:

   ```bash
   sudo systemctl enable --now haproxy
   ```

## Configure Master Keepalived

On the master load balancer VM, configure Keepalived to manage the VIP.

1. Update the Keepalived configuration:

   ```bash
   cat <<EOF > /etc/keepalived/keepalived.conf
   global_defs {
      enable_script_security
      script_user root
   }

   vrrp_script check_haproxy {
      script "killall -0 haproxy"
      interval 2
      weight 2
   }

   vrrp_instance KUBE_LB {
      state MASTER
      interface ens192
      virtual_router_id 51
      priority 101
      virtual_ipaddress {
         ${vip}/32
      }
      track_script {
         check_haproxy
      }
   }
   EOF
   ```

1. Verify the configuration:

   ```bash
   sudo keepalived -t -l -f /etc/keepalived/keepalived.conf
   ```

1. Enable and start Keepalived:

   ```bash
   sudo systemctl enable --now keepalived
   ```

## Configure Backup Keepalived

On the backup load balancer VM, configure Keepalived.

1. Update the Keepalived configuration:

   ```bash
   cat <<EOF > /etc/keepalived/keepalived.conf
   global_defs {
      enable_script_security
      script_user root
   }

   vrrp_script check_haproxy {
      script "killall -0 haproxy"
      interval 2
      weight 2
   }

   vrrp_instance KUBE_LB {
      state BACKUP
      interface ens192
      virtual_router_id 51
      priority 100
      virtual_ipaddress {
         ${vip}/32
      }
      track_script {
         check_haproxy
      }
   }
   EOF
   ```

1. Verify the configuration:

   ```bash
   sudo keepalived -t -l -f /etc/keepalived/keepalived.conf
   ```

1. Enable and start Keepalived:

   ```bash
   sudo systemctl enable --now keepalived
   ```

## Verify Configuration

1. Check HAProxy status on both VMs:

   ```bash
   sudo systemctl status haproxy
   ss -ntlp | grep -E '80|443|8000|6443'
   ```

1. Verify the VIP on the master VM:

   ```bash
   ip a | grep ${vip}/32
   ```

   The VIP should appear on the master and fail over to the backup if Keepalived is stopped on the master.

1. Test API server accessibility via the VIP from a Kubernetes node (After kubernetes installation):

   ```bash
   curl -k https://${vip}:6443/version
   ```

   Expected output: Kubernetes version information.

1. Test ingress connectivity via the VIP (After Ingress installation):

   ```bash
   curl http://${vip}
   curl -k https://${vip}
   ```

   Expected output depends on your ingress controller setup (e.g., NGINX Ingress).

## Next Steps

- Proceed to [kubespray-install.md](kubespray-install.md) to deploy the Kubernetes cluster.
- For ongoing cluster management, see [kubespray-maintenance.md](kubespray-maintenance.md).
- Verify all steps with [checklist.md](checklist.md).

---

**Keywords**: Kubernetes load balancer, HAProxy Keepalived setup, Kubernetes ingress load balancing, high availability API server, Kubernetes firewall configuration, DevOps, Kubespray, air-gapped, online deployment, Ansible, RHEL, Rocky Linux
