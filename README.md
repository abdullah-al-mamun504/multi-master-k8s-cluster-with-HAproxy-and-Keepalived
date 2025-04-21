# Kubernetes High Availability Cluster Setup Guide

This guide provides instructions for setting up a high-availability Kubernetes cluster with multiple control plane nodes and a load balancer configuration.

## Architecture Overview

- **Load Balancer**: HAProxy with Keepalived for high availability
- **Control Plane Nodes**: Multiple nodes for redundancy
- **Worker Nodes**: For running workloads

## Prerequisites

- Multiple Ubuntu servers (minimum 4: 3 control plane + 1 worker)
- Root or sudo access on all nodes
- Stable network connectivity between all nodes
- Unique hostnames, MAC addresses, and product_uuids for each node

## 1. Pre-requisite Configuration for All Nodes

Run this script on **all nodes** (both control plane and worker nodes):

```bash
#!/bin/sh

# Update system
apt-get update && apt-get upgrade -y

# Disable swap
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# Enable necessary kernel modules
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# Configure sysctl settings for Kubernetes networking
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# Install required dependencies
apt-get install -y containerd apt-transport-https ca-certificates curl gpg

# Configure containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable --now containerd

# Add Kubernetes repository
mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update

# Install Kubernetes tools
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable --now kubelet
```

## 2. Load Balancer Setup (HAProxy & Keepalived)

### 2.1. Install HAProxy and Keepalived

```bash
apt-get install -y haproxy keepalived
```

### 2.2. Configure HAProxy

Edit `/etc/haproxy/haproxy.cfg`:

```
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

# Kubernetes API Server Load Balancing
frontend kubernetes
        bind *:6443
        mode tcp
        option tcplog
        default_backend k8s-masters

backend k8s-masters
        mode tcp
        balance roundrobin
        option tcp-check
        server cp-1 192.168.152.128:6443 check
        server cp-2 192.168.152.129:6443 check
        server cp-3 192.168.152.130:6443 check
```

> **Note**: Update the IP addresses of the control plane nodes according to your environment.

### 2.3. Configure Keepalived

Create or edit `/etc/keepalived/keepalived.conf`:

```
vrrp_instance VI_1 {
    state MASTER
    interface ens32          # Change this to match your network interface
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass secretpassword
    }
    virtual_ipaddress {
        192.168.152.200     # Virtual IP address for the load balancer
    }
}
```

> **Note**: Change the interface name and virtual IP address according to your network setup.

### 2.4. Start and Enable Services

```bash
systemctl restart haproxy
systemctl enable haproxy
systemctl restart keepalived
systemctl enable keepalived
```

## 3. Initialize Kubernetes Control Plane

Run the following commands on the **first control plane node**:

### 3.1. Pull Required Images

```bash
kubeadm config images pull
```

### 3.2. Initialize the Cluster

```bash
kubeadm init --control-plane-endpoint "192.168.152.200:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
```

> **Note**: Replace the IP address with your virtual IP configured in Keepalived.

### 3.3. Set Up kubectl Access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3.4. Join Additional Control Plane Nodes

From the output of the `kubeadm init` command, use the control plane join command on your other control plane nodes:

```bash
kubeadm join 192.168.152.200:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <certificate-key>
```

## 4. Install Network Plugin (CNI)

After initializing the cluster, install a Container Network Interface (CNI) like Calico:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

## 5. Join Worker Nodes

Use the worker node join command from the output of `kubeadm init`:

```bash
kubeadm join 192.168.152.200:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

## 6. Verify Cluster Status

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

## Troubleshooting

- If tokens expire, create new ones:
  ```bash
  kubeadm token create --print-join-command
  ```

- To generate a new certificate key for control plane join:
  ```bash
  kubeadm init phase upload-certs --upload-certs
  ```

- Check logs for issues:
  ```bash
  journalctl -xeu kubelet
  ```

## Security Considerations

- Change the default keepalived password
- Implement network policies
- Use RBAC for access control
- Consider encrypting etcd data
