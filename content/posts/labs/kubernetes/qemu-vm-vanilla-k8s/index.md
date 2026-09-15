---
title: "How to Install Kubernetes in a QEMU VM?"
date: 2026-09-15T19:43:12+03:00
description: "Step-by-step guide to installing QEMU on Linux Mint and running a single-node Kubernetes cluster inside an Ubuntu 26.04 VM."
menu:
  sidebar:
    name: qemu-vm-k8s
    identifier: qemu-vm-k8s
    parent: kubernetes
    weight: 10
hero: /images/sections/posts/hero-kubernetes-qemu.jpeg
tags:
  - kubernetes
  - qemu
  - ubuntu-26-04
categories:
  - Basic
created: 2026-09-15 19:43:54
updated: 2026-09-15 19:44:12
---

**TL;DR** — This guide walks through installing QEMU on Linux Mint, creating an Ubuntu 26.04 Server VM with dual networking, and bootstrapping a single-node Kubernetes cluster inside it.

## Why

I have been doing all my virtualization work with [Oracle VirtualBox](https://www.virtualbox.org/). It works great, but it is time to switch to a more professional tool: [QEMU](https://www.qemu.org/).

QEMU can do everything VirtualBox does and more. It can emulate ARM architecture on x86, offers better performance, and is fully open source. In this guide I will create a VM with Ubuntu 26.04 and bootstrap a Kubernetes cluster inside it.

## Prerequisites

- A Debian-based Linux machine (guide uses Linux Mint)
- Internet access
- KVM-capable CPU (`ls -l /dev/kvm` should return a device)

## Step 1: Install QEMU

Since I am using Linux Mint, `apt` is my package manager. All commands are `apt`-based. You can find the `dnf`, `pacman`, `apk`, or `yum` equivalent on the [official QEMU download page](https://www.qemu.org/download/).

```bash
# Full system emulation (needed for this guide)
sudo apt-get install -y qemu-system

# Optional: user-space Linux binary emulation
sudo apt-get install -y qemu-user-static
```

> **Note:**: As of 15 September 2026, the latest QEMU release is **11.1.1**, but `apt` installs an older version. This is a typical Debian packaging delay — it is fine for our purposes.

## Step 2: Create an Ubuntu VM

I will create an [Ubuntu Server 26.04.1 LTS](https://releases.ubuntu.com/26.04.1/ubuntu-26.04.1-live-server-amd64.iso) VM with **two network adapters** — one NAT and one host-only (bridged). Two adapters are the best setup for a Kubernetes node because:

- **Clean separation of data planes** — The NAT adapter handles outbound internet traffic (package updates, container image pulls), while the bridge adapter provides a stable, host-accessible IP for the Kubernetes API server, Ingress controllers, and NodePort services, without mixing routing tables.
- **No port-forwarding bottlenecks** — With a dedicated host-accessible IP you do not need to manually map every port each time you deploy a new workload.

### Create a Virtual Bridge

Run these commands once on the **host machine** to create a bridge the VM will attach to.

```bash
# On Host Machine — create an isolated bridge
sudo ip link add br-k8s type bridge
sudo ip addr add 192.168.100.1/24 dev br-k8s
sudo ip link set br-k8s up

# Allow QEMU's bridge helper to use this bridge
sudo mkdir -p /etc/qemu
echo "allow br-k8s" | sudo tee -a /etc/qemu/bridge.conf
# NOTE: The helper path may differ on other distros
sudo chmod u+s /usr/lib/qemu/qemu-bridge-helper
```

### Create the Disk and Install Ubuntu

VM files will live at `/home/taygun/Vms/Kubernetes-Test-Ubuntu-26`.

```bash
# On Host Machine
mkdir -p /home/taygun/Vms/Kubernetes-Test-Ubuntu-26
# Create a shared folder
mkdir -p /home/taygun/Vms/Kubernetes-Test-Ubuntu-26/shared
cd /home/taygun/Vms/Kubernetes-Test-Ubuntu-26

# Create 80G Disk
# On Host Machine
qemu-img create -f qcow2 disk.qcow2 80G
```

Verify KVM is available. Without it the VM falls back to slow software emulation (TCG):

```bash
# On Host Machine
ls -l /dev/kvm
# crw-rw----+ 1 root kvm 10, 232 Sep 15 05:58 /dev/kvm
```

Download `ubuntu-26.04.1-live-server-amd64.iso` from [Ubuntu releases](https://releases.ubuntu.com/26.04.1/ubuntu-26.04.1-live-server-amd64.iso) into `/home/taygun/Vms/iso/`, then run the **installation** command below.

> **Note:** The `-cdrom` flag is only needed during installation. Remove it for subsequent boots.

```bash
# On Host Machine — INSTALLATION boot (includes -cdrom)
qemu-system-x86_64 \
  -name k8s-test-01 \
  -machine q35,accel=kvm \
  -cpu host \
  -smp 4,cores=4,threads=1 \
  -m 4G \
  -drive file=/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/disk.qcow2,if=virtio,format=qcow2 \
  -cdrom /home/taygun/Vms/iso/ubuntu-26.04.1-live-server-amd64.iso \
  -boot order=dc \
  -netdev user,id=nat0 \
  -device virtio-net-pci,netdev=nat0,mac=52:54:00:aa:bb:01 \
  -netdev bridge,id=host0,br=br-k8s \
  -device virtio-net-pci,netdev=host0,mac=52:54:00:aa:bb:02 \
  -vga virtio \
  -display gtk \
  -rtc base=utc \
  -pidfile /home/taygun/Vms/Kubernetes-Test-Ubuntu-26/vm.pid
```

**Flag reference:**

| Flag | Description |
|------|-------------|
| `-name k8s-test-01` | Display name of the VM |
| `-machine q35,accel=kvm` | Modern Intel Q35 chipset with KVM hardware acceleration |
| `-cpu host` | Exposes the host's exact CPU features for near-native performance |
| `-smp 4,cores=4,threads=1` | 4 vCPUs (4 physical cores, 1 thread each) |
| `-m 4G` | 4 GiB RAM |
| `-drive file=...,if=virtio,format=qcow2` | High-performance VirtIO disk backed by a QCOW2 image |
| `-cdrom ...` | Ubuntu ISO mounted as an optical drive (installation only) |
| `-boot order=dc` | Boot from optical drive first, then hard drive |
| `-netdev user,id=nat0` | User-mode NAT network (no root privileges needed for outbound access) |
| `-device virtio-net-pci,netdev=nat0,mac=...` | VirtIO NIC attached to the NAT backend |
| `-netdev bridge,id=host0,br=br-k8s` | Network backend bridged to the host's `br-k8s` interface |
| `-device virtio-net-pci,netdev=host0,mac=...` | VirtIO NIC attached to the bridge (cluster traffic) |
| `-vga virtio` | Para-virtualized VirtIO GPU for smooth display |
| `-display gtk` | Opens the VM in a local GTK window |
| `-rtc base=utc` | Sets guest RTC to UTC to avoid time-drift issues |
| `-pidfile ...` | Writes the VM PID to a file for scripted control |

Follow the Ubuntu installer using the defaults. When prompted, **enable OpenSSH Server**. After installation completes, select **"Reboot Now"** and then quit QEMU.

### First Boot (Without ISO)

Start the VM again — this time without the `-cdrom` flag but with the shared folder:

```bash
# On Host Machine — first normal boot
qemu-system-x86_64 \
  -name k8s-test-01 \
  -machine q35,accel=kvm \
  -cpu host \
  -smp 4,cores=4,threads=1 \
  -m 4G \
  -drive file=/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/disk.qcow2,if=virtio,format=qcow2 \
  -boot order=c \
  -netdev user,id=nat0 \
  -device virtio-net-pci,netdev=nat0,mac=52:54:00:aa:bb:01 \
  -netdev bridge,id=host0,br=br-k8s \
  -device virtio-net-pci,netdev=host0,mac=52:54:00:aa:bb:02 \
  -virtfs local,path=/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/shared,mount_tag=hostshare,security_model=mapped-xattr \
  -vga virtio \
  -display gtk \
  -rtc base=utc \
  -pidfile /home/taygun/Vms/Kubernetes-Test-Ubuntu-26/vm.pid
```

The boot may take a minute or two while `systemd-networkd-wait-online` and `cloud-init-network.service` time out. This is expected and will be fixed in the next step.

### Configure Static Networking Inside the VM

Log in at the VM console (copy-paste is not available here, so type carefully).

**Disable cloud-init network management:**

```bash
# On VM
sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg <<'EOF'
network: {config: disabled}
EOF
```

Edit the Netplan configuration

```bash
# On VM
vi /etc/netplan/00-installer-config.yaml
```

Press `dG` to delete all existing content, then type the following YAML manually:

```yaml
# This is the network config written by 'subiquity'
network:
  version: 2
  ethernets:
    nat0:
      match:
        macaddress: "52:54:00:aa:bb:01"
      set-name: nat0
      dhcp4: true
      dhcp4-overrides:
        use-dns: true
        route-metric: 100

    host0:
      match:
        macaddress: "52:54:00:aa:bb:02"
      set-name: host0
      dhcp4: false
      accept-ra: false
      addresses:
        - 192.168.100.10/24
      optional: true
      routes:
        - to: 192.168.100.0/24
          via: 0.0.0.0
          scope: link
```

Apply the netplan and reboot.

```bash
# On VM
sudo netplan generate
sudo netplan apply
sudo reboot
```

After rebooting, the login prompt should appear immediately. Log in and verify connectivity:

```bash
# On VM
ip -br addr show
ip route
sudo apt update
```

### Switch to Headless Mode with SSH Access

Now that networking is working, restart QEMU in headless (daemon) mode with SSH port forwarding. Add the following aliases to `~/.zshrc` (or `~/.bashrc`) on the **host**:

```bash
# On Host Machine
tee -a ~/.zshrc > /dev/null <<'EOF'
# ========
alias vm-k8s-up='qemu-system-x86_64 \
  -name k8s-test-01 \
  -machine q35,accel=kvm \
  -cpu host \
  -smp 4,cores=4,threads=1 \
  -m 4G \
  -drive file=/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/disk.qcow2,if=virtio,format=qcow2 \
  -boot order=c \
  -netdev user,id=nat0,hostfwd=tcp::2202-:22 \
  -device virtio-net-pci,netdev=nat0,mac=52:54:00:aa:bb:01 \
  -netdev bridge,id=host0,br=br-k8s \
  -device virtio-net-pci,netdev=host0,mac=52:54:00:aa:bb:02 \
  -virtfs local,path=/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/shared,mount_tag=hostshare,security_model=mapped-xattr \
  -display none \
  -daemonize \
  -rtc base=utc \
  -monitor unix:/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/vm.monitor,server=on,wait=off \
  -pidfile /home/taygun/Vms/Kubernetes-Test-Ubuntu-26/vm.pid'
alias vm-k8s-down='echo "system_powerdown" | socat - UNIX-CONNECT:/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/vm.monitor'
alias vm-k8s-status='
PIDFILE=/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/vm.pid
if [ -f "$PIDFILE" ] && kill -0 "$(cat "$PIDFILE")" 2>/dev/null; then
    echo "VM: RUNNING"
    echo "PID: $(cat "$PIDFILE")"
    nc -z localhost 2202 2>/dev/null && echo "SSH: READY (localhost:2202)" || echo "SSH: NOT READY"
else
    echo "VM: STOPPED"
fi'
alias vm-k8s-snapshot-ls='qemu-img snapshot -l /home/taygun/Vms/Kubernetes-Test-Ubuntu-26/disk.qcow2'
alias ssh-k8s-test-01='ssh k8s-test-01'
# ========
EOF
# Add SSH client config — change "User" to your VM username
tee -a ~/.ssh/config > /dev/null <<'EOF'
Host k8s-test-01
    HostName localhost
    Port 2202
    User taygun
    IdentitiesOnly yes
EOF
source ~/.zshrc
```

Test that everything works end-to-end:

```bash
vm-k8s-up
vm-k8s-status
# VM: RUNNING
# PID: 97376
# SSH: READY (localhost:2202)
ssh-k8s-test-01 'ls -l /home/'
# taygun@localhost's password:
# total 4
# drwxr-x--- 4 taygun taygun 4096 Sep 15 08:17 taygun
vm-k8s-down
# QEMU 8.2.2 monitor - type 'help' for more information
# (qemu) system_powerdown
# (qemu) %
vm-k8s-status
# VM: STOPPED
```

The VM is now ready for Kubernetes.

## Step 3: Bootstrap Kubernetes

SSH into the VM (`ssh-k8s-test-01` or `vm-k8s-up && ssh-k8s-test-01`) and run the following blocks in order.

```bash
# On VM

# ========
## OS Preparation
# Disable swap — required by kubelet
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Set a meaningful hostname for the control-plane node
sudo hostnamectl set-hostname k8s-control-01

# Load required kernel modules for container networking
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Enable IP forwarding and bridge-netfilter iptables rules — required for pod networking
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
# ========

# ========
## Install containerd
sudo apt update
sudo apt install -y containerd
# Generate default config, then switch to systemd cgroup driver —
# this must match kubelet's cgroup driver or kubelet will fail to start
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
# ========

# ========
## Install kubeadm, kubelet, kubectl (version v1.37)
sudo apt install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes apt repo and signing key (adjust version if needed)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.37/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.37/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl

# Pin the versions to prevent accidental upgrades that could break cluster compatibility
sudo apt-mark hold kubelet kubeadm kubectl
# ========

# ========
## Initialize the cluster
# --apiserver-advertise-address: the static IP of the host0 bridge interface
# --pod-network-cidr: must match the CIDR you pass to the CNI plugin (Cilium below)
sudo kubeadm init \
  --apiserver-advertise-address=192.168.100.10 \
  --pod-network-cidr=10.244.0.0/16
# ...
# kubeadm join 192.168.100.10:6443 --token dgsqc9.8of7nv1ryp5pyin2 \
#    --discovery-token-ca-cert-hash sha256:94e8fd9815aea744fa9659e4c421fc9fc1c29bf7104e5c75e3493775952263ee
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# CoreDNS pods will be Pending until a CNI is installed — that is expected
kubectl get pods -A
# NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
# kube-system   coredns-559f6c778d-2lgq9                 0/1     Pending   0          58s
# kube-system   coredns-559f6c778d-ndmvn                 0/1     Pending   0          58s
# kube-system   etcd-k8s-control-01                      1/1     Running   0          66s
# kube-system   kube-apiserver-k8s-control-01            1/1     Running   0          66s
# kube-system   kube-controller-manager-k8s-control-01   1/1     Running   0          66s
# kube-system   kube-proxy-dvz85                         1/1     Running   0          58s
# kube-system   kube-scheduler-k8s-control-01            1/1     Running   0          66s
# ========

# ========
## Install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
# ========

# ========
## Install Cilium as the CNI
helm repo add cilium https://helm.cilium.io/
helm repo update
# Install Cilium as the CNI, matching the pod CIDR used in kubeadm init
helm install cilium cilium/cilium \
  --version 1.20.1 \
  --namespace kube-system \
  --set ipam.operator.clusterPoolIPv4PodCIDRList="10.244.0.0/16" \
  --set kubeProxyReplacement=false \
  --set operator.replicas=1
# ========

# ========
## Post-install tuning
# A single-node cluster only needs one CoreDNS replica
kubectl scale deploy -n kube-system coredns --replicas=1
# deployment.apps/coredns scaled
kubectl get pods -A && kubectl get nodes
# NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
# kube-system   cilium-envoy-p7q9s                       1/1     Running   0          17m
# kube-system   cilium-lzzq9                             1/1     Running   0          17m
# kube-system   cilium-operator-cd976bbf6-56kfd          1/1     Running   0          17m
# kube-system   coredns-559f6c778d-ndmvn                 1/1     Running   0          22m
# kube-system   etcd-k8s-control-01                      1/1     Running   0          23m
# kube-system   kube-apiserver-k8s-control-01            1/1     Running   0          23m
# kube-system   kube-controller-manager-k8s-control-01   1/1     Running   0          23m
# kube-system   kube-proxy-dvz85                         1/1     Running   0          22m
# kube-system   kube-scheduler-k8s-control-01            1/1     Running   0          23m
# NAME             STATUS   ROLES           AGE   VERSION
# k8s-control-01   Ready    control-plane   23m   v1.37.0
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
# node/k8s-control-01 untainted
# ========
```

Your VM and Kubernetes cluster are ready for deployments.

### Copy kubeconfig to the host (Optional)

To manage the cluster from your host machine without SSH-ing into the VM:

```bash
## On Host Machine
scp k8s-test-01:~/.kube/config ~/.kube/config-k8s-test
kubectl --kubeconfig ~/.kube/config-k8s-test get pods -A
# NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
# kube-system   cilium-envoy-p7q9s                       1/1     Running   0          49m
# kube-system   cilium-lzzq9                             1/1     Running   0          49m
# kube-system   cilium-operator-cd976bbf6-56kfd          1/1     Running   0          49m
# kube-system   coredns-559f6c778d-ndmvn                 1/1     Running   0          54m
# kube-system   etcd-k8s-control-01                      1/1     Running   0          54m
# kube-system   kube-apiserver-k8s-control-01            1/1     Running   0          54m
# kube-system   kube-controller-manager-k8s-control-01   1/1     Running   0          54m
# kube-system   kube-proxy-dvz85                         1/1     Running   0          54m
# kube-system   kube-scheduler-k8s-control-01            1/1     Running   0          54m
```

## Step 4: Snapshot and Restore

First shut down the vm.

```bash
## On Host Machine
vm-k8s-down && sleep 10
vm-k8s-status
# VM: STOPPED
```

### Snapshot

Create a snapshot.

```bash
## On Host Machine
qemu-img snapshot -c fresh-k8s \
  /home/taygun/Vms/Kubernetes-Test-Ubuntu-26/disk.qcow2
vm-k8s-snapshot-ls
# Snapshot list:
# ID        TAG               VM SIZE                DATE     VM CLOCK     ICOUNT
# 1         fresh-k8s             0 B 2026-09-15 13:16:21 00:00:00.000          0
```

To delete a snapshot:

```bash
## On Host Machine — CAUTION: this is irreversible
qemu-img snapshot -d fresh-k8s \
  /home/taygun/Vms/Kubernetes-Test-Ubuntu-26/disk.qcow2
```

### Restore on Another Machine (e.g., a Laptop)

Transfer `/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/disk.qcow2` to the target machine via `scp` or a bootable USB stick, then repeat the bridge setup:

```bash
# On Laptop Host Machine
sudo ip link add br-k8s type bridge
sudo ip addr add 192.168.100.1/24 dev br-k8s
sudo ip link set br-k8s up
sudo mkdir /etc/qemu/
echo "allow br-k8s" | sudo tee -a /etc/qemu/bridge.conf
sudo chmod u+s /usr/lib/qemu/qemu-bridge-helper
```

Start the VM (same headless command as the `vm-k8s-up` alias):

```bash
# On Laptop Host Machine
qemu-system-x86_64 \
  -name k8s-test-01 \
  -machine q35,accel=kvm \
  -cpu host \
  -smp 4,cores=4,threads=1 \
  -m 4G \
  -drive file=/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/disk.qcow2,if=virtio,format=qcow2 \
  -boot order=c \
  -netdev user,id=nat0,hostfwd=tcp::2202-:22 \
  -device virtio-net-pci,netdev=nat0,mac=52:54:00:aa:bb:01 \
  -netdev bridge,id=host0,br=br-k8s \
  -device virtio-net-pci,netdev=host0,mac=52:54:00:aa:bb:02 \
  -virtfs local,path=/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/shared,mount_tag=hostshare,security_model=mapped-xattr \
  -display none \
  -daemonize \
  -rtc base=utc \
  -monitor unix:/home/taygun/Vms/Kubernetes-Test-Ubuntu-26/vm.monitor,server=on,wait=off \
  -pidfile /home/taygun/Vms/Kubernetes-Test-Ubuntu-26/vm.pid
```

> **Tip:** Also copy `~/.ssh/config` and `~/.zshrc` from the original host to the laptop so the `vm-k8s-*` and `ssh-k8s-test-01` aliases are available.

SSH into the VM and bring the cluster back up:

```bash
# On VM (via ssh-k8s-test-01)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
# Restart kubelet and containerd
sudo systemctl restart containerd kubelet
kubectl get pods -A
# NAMESPACE     NAME                                     READY   STATUS    RESTARTS        AGE
# kube-system   cilium-envoy-p7q9s                       1/1     Running   1 (5h43m ago)   6h49m
# kube-system   cilium-lzzq9                             1/1     Running   1 (5h43m ago)   6h49m
# kube-system   cilium-operator-cd976bbf6-56kfd          1/1     Running   1 (5h43m ago)   6h49m
# kube-system   coredns-559f6c778d-ndmvn                 1/1     Running   1 (5h43m ago)   6h54m
# kube-system   etcd-k8s-control-01                      1/1     Running   1 (5h43m ago)   6h54m
# kube-system   kube-apiserver-k8s-control-01            1/1     Running   1 (5h43m ago)   6h54m
# kube-system   kube-controller-manager-k8s-control-01   1/1     Running   1 (5h43m ago)   6h54m
# kube-system   kube-proxy-dvz85                         1/1     Running   1 (5h43m ago)   6h54m
# kube-system   kube-scheduler-k8s-control-01            1/1     Running   1 (5h43m ago)   6h54m
```
