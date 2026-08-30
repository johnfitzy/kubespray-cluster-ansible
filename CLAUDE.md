# kubernetes-cluster-ansible — Claude Context

Personal Ansible project that provisions a Kubernetes cluster with **Kubespray**
across multiple environments. Migrated from k3s to Kubespray (June 2026).

## Purpose

Stand up a **base** Kubernetes cluster (Calico CNI, kube-proxy IPVS) ready for
workloads to be layered on later via separate playbooks. The first such future
playbook is **MinIO + DirectPV**, which will consume the raw data disks.

## Environments

- **Vagrant** (`inventory/local_vagrant/`): 3x Ubuntu 24.04 VMs on 192.168.56.0/24
  (knode1/knode2/knode3). Low-resource laptop lab — **no HA**. Box is
  `bento/ubuntu-24.04` (packer box w/ Guest Additions); the project dir is
  synced to `/vagrant` in each guest. NIC names are `eth0` (NAT) / `eth1`
  (192.168.56.x host-only). VirtualBox provider.
- **Home lab** (`inventory/home_lab/`, coming soon): 3x Lenovo m910q.

## Topology (local_vagrant)

| Node   | IP             | Kubespray groups                         |
|--------|----------------|------------------------------------------|
| knode1 | 192.168.56.111 | kube_control_plane, etcd, kube_node      |
| knode2 | 192.168.56.112 | kube_node                                |
| knode3 | 192.168.56.113 | kube_node                                |

Single control-plane + etcd (knode1), which is also a worker. All three are
`kube_node`.

## Playbook Flow

```
site.yml
 ├── pre-kubespray.yml        # roles: base, geerlingguy.security
 └── kubespray/cluster.yml    # the Kubernetes deploy (submodule) — currently commented out
```

- `vagrant up` / `vagrant provision` runs `site.yml` against all nodes (Ansible
  provisioner is pinned to the **last** VM, `ansible.limit = "all"`, inventory
  `inventory/local_vagrant`). Ansible runs on the **host** — activate `venv/`
  first: `source venv/bin/activate && vagrant provision`.
- Runs standalone too: `ansible-playbook -i inventory/local_vagrant site.yml`.
- `kubespray/cluster.yml` import in `site.yml` is commented out until the base
  layer is verified.

## Kubespray

- Vendored as a **git submodule pinned to `v2.31.0`** (Kubernetes ~v1.35) at `kubespray/`.
- Pins `ansible==11.13.0` (ansible-core 2.18) via `kubespray/requirements.txt` —
  install into the project `venv/`. That package bundles all needed collections
  (incl. `ansible.posix`, `community.general` used by the base roles).
- `ansible.cfg` wires `roles_path` to `./roles:./kubespray/roles` and `library`
  to `./kubespray/library:./kubespray/plugins/modules`.

### group_vars (inventory/local_vagrant/group_vars)

Seeded from the v2.31.0 sample, with overrides:
- `kube_network_plugin: calico` (standard dataplane, **not** eBPF)
- `kube_proxy_mode: ipvs`, `kube_proxy_strict_arp: true`
- `helm_enabled: true`, `metrics_server_enabled: true`
- `kubeconfig_localhost: true`, `kubectl_localhost: true` → admin kubeconfig
  lands at `inventory/local_vagrant/artifacts/admin.conf`
- `all/base.yml` holds user accounts (`user_config`, `k8s_admins`) +
  geerlingguy.security vars — **not** a kubespray file, kept separate.

## Disks / Storage

- Vagrant attaches **two raw 10GB disks per node** as `sdb`/`sdc`, on the
  box's existing `SATA Controller` (AHCI) at ports 1-2 — the OS disk is `sda`
  on port 0. They are **left completely raw** — no partition, no mkfs.
- Note: the disks must go on the box's *own* OS-disk controller, not a
  separate added controller. `cloud-image/ubuntu-24.04` (previous box) put its
  OS disk on VirtIO-SCSI, and adding a SATA controller made the VirtualBox
  BIOS halt with "Could not read from the boot medium!" (it probes AHCI before
  VirtIO-SCSI). `bento/ubuntu-24.04` already uses `SATA Controller` for the OS
  disk, so extra disks just take higher ports on it.
- **No storage layer is installed by this repo.** Storage is deferred to a
  future MinIO + DirectPV playbook. DirectPV discovers and formats the raw
  drives itself; MinIO does its own erasure coding.
- Background: OpenEBS was considered and rejected — its whole-disk path
  (NDM / Local PV Device) is archived and gone from OpenEBS 4.x, and MinIO
  recommends DirectPV (or raw local drives), not OpenEBS.

## Roles

- `base`: creates `developer` group, user accounts, SSH keys, `k8s-admin` group.
- `geerlingguy.security` (v3.0.0, via `roles/requirements.yml`): SSH hardening,
  fail2ban, unattended-upgrades. Does **not** configure UFW. Ubuntu 24.04 ships
  UFW disabled, so no firewall rules are enforced by default in the lab.

## Cluster networking ports (between nodes)

- 6443/TCP — kube-apiserver
- 2379-2380/TCP — etcd
- 10250/TCP — kubelet
- 179/TCP — Calico BGP (if BGP mode); Calico defaults to VXLAN otherwise
- 4789/UDP — Calico VXLAN

## Known TODOs

- knode1 verified booting on `bento/ubuntu-24.04` (raw disks land on `sdb`/`sdc`
  as expected, `/vagrant` synced). Still need to bring up knode2/knode3 and run
  the cluster end-to-end.
- Home lab inventory not yet created.
- Future: separate MinIO + DirectPV playbook consuming sdb/sdc.
