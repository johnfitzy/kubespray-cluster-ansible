# Kubernetes Cluster Ansible

## Purpose

Build a Kubernetes cluster on VMs (and, later, bare metal) using
[Kubespray](https://kubespray.io), deployable to multiple environments.

The cluster is intentionally a **base environment only**: Kubernetes with the
Calico CNI (kube-proxy in IPVS mode) and nothing else of consequence. Each node
is given two extra raw 10GB disks that are **left untouched**; they are
reserved for a future, separate MinIO.

### Environments

- **Vagrant/VirtualBox** (`inventory/local_vagrant/`) — 4x Ubuntu 24.04 VMs (1 control-plane + 3 workers)
- **Home lab** (coming soon) — 3x Lenovo m910q
- Future: cheap cloud VMs

### Topology (local Vagrant lab)

Low-resource laptop demo — **no HA**:

| Node   | IP             | Roles                                |
|--------|----------------|--------------------------------------|
| cp1  | 192.168.56.111 | control-plane + etcd (only)          |
| kn1  | 192.168.56.112 | worker                               |
| kn2  | 192.168.56.113 | worker                               |
| kn3  | 192.168.56.114 | worker                               |

cp1 is control-plane only (tainted). The three workers back future storage
workloads: 3 nodes × 2 disks.

---

## Quickstart

```bash
git submodule update --init --recursive      # already present, but for fresh clones
python3 -m venv venv && source venv/bin/activate
pip install -U pip && pip install -r kubespray/requirements.txt
ansible-galaxy install -r roles/requirements.yml
source venv/bin/activate && vagrant up        # venv MUST be active for the provisioner
export KUBECONFIG=inventory/local_vagrant/artifacts/admin.conf
kubectl get nodes -o wide
```

See the sections below for detail.

---

## Kubespray

Kubespray is vendored as a **git submodule pinned to `v2.31.0`** (Kubernetes
~v1.35) under `kubespray/`. The project's `ansible.cfg`, `site.yml`, and
inventory are wired to drive it.

```
site.yml
 ├── pre-kubespray.yml        # base users/SSH + geerlingguy.security hardening
 └── kubespray/cluster.yml    # the Kubernetes deploy
```

Cluster configuration lives in `inventory/local_vagrant/group_vars/`, seeded
from the v2.31.0 sample with these deliberate overrides:

- `kube_network_plugin: calico` (standard dataplane, **not** eBPF)
- `kube_proxy_mode: ipvs`, `kube_proxy_strict_arp: true`
- `helm_enabled: true`, `metrics_server_enabled: true`
- `kubeconfig_localhost: true`, `kubectl_localhost: true`

---

## Setup

### 1. Clone with submodules

```shell
git submodule update --init --recursive
```

### 2. Create the Python venv (Kubespray pins its own Ansible)

```shell
python3 -m venv venv
source venv/bin/activate
pip install -U pip
pip install -r kubespray/requirements.txt
```

This installs `ansible==11.13.0` (ansible-core 2.18) plus all collections
Kubespray and the base roles need.

### 3. Install the Galaxy role(s)

```shell
ansible-galaxy install -r roles/requirements.yml
```

### 4. (Optional) hosts file

```shell
192.168.56.111 cp1
192.168.56.112 kn1
192.168.56.113 kn2
192.168.56.114 kn3
```

---

## Run

> The Vagrant `ansible` provisioner runs Kubespray, so the **venv must be
> active** in the shell you run `vagrant up` from.

```shell
source venv/bin/activate
vagrant up
```

This boots the three VMs (each with two raw 10GB data disks) and runs `site.yml`
against all of them.

### Run the playbook manually (re-runs / iteration)

```shell
source venv/bin/activate
ansible-playbook -b -e "ansible_user=john" -e ansible_ssh_private_key_file=/home/john/.ssh/kubespray-ansible-cluster -i inventory/local_vagrant/ site.yml
```

### Access the cluster

Kubespray copies the admin kubeconfig to the inventory's `artifacts/` dir:

```shell
export KUBECONFIG=inventory/local_vagrant/artifacts/admin.conf
kubectl get nodes -o wide
```

### Verify the raw disks (for the future MinIO/DirectPV work)

```shell
vagrant ssh kn1 -c "lsblk -dn -o NAME,SIZE,TYPE,FSTYPE"   # any worker
# expect sdb / sdc at 10G with no partitions and no filesystem
```

### Tear down

```shell
vagrant destroy -f
```