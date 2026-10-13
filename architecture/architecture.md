# Architecture
This is the proposed architecture of this lab. Note that multiple services could be groups in one single VM. This separation I performed is for learning purposes only.

The bastion is only the point of configuration: it holds the Ansible playbooks and the lab CA, and it routes the isolated OCP network to the internet. Every lab service runs on its own VM.

## Domain
lab.local

## Nodes and functions

| Hostname | CPU | RAM | DISK | OS | IP | Purpose |
|----------|-----|-----|------|----|----|---------|
| bastion | 4 | 4 GB | 60 GB | RedHat 9 | 10.0.0.50 (home) <br> 10.10.0.2 (ocp) | Ansible host, router/NAT for the OCP network, lab CA |
| dhcp | 1 | 2 GB | 50 GB | RedHat 9 | 10.10.0.3 | DHCP server (dhcpd) |
| dns | 1 | 2 GB | 50 GB | RedHat 9 | 10.10.0.4 | DNS server (bind) |
| registry | 2 | 8 GB | 300 GB | RedHat 9 | 10.10.0.5 | mirror registry (Quay) |

The VMs are defined in `installation/ansible-playbooks/vars/main.yaml` (`proxmox_vms`). Keep both in sync.

## Reserved addresses

| IP | Name | Purpose |
|----|------|---------|
| 10.10.0.1 | - | free (the bastion is the gateway at .2) |
| 10.10.0.10 | api.ocp4 / api-int.ocp4 | cluster API |
| 10.10.0.11 | *.apps.ocp4 | cluster ingress |
| 10.10.0.21-23 | master01-03.ocp4 | control plane nodes |
| 10.10.0.31-32 | worker01-02.ocp4 | worker nodes |
| 10.10.0.200-250 | - | DHCP pool |

The cluster addresses are planned; they are already in the DNS records (`dns_ocp_records`) but the VMs do not exist yet.

## Network connections

The OCP network (10.10.0.0/24) is isolated on the `ocp` bridge. The bastion is the only node with two NICs: it forwards and NATs this network to the home network, so every VM reaches the internet through 10.10.0.2. The nodes resolve names against the DNS server (10.10.0.4), which forwards public names upstream.

Still to be placed on a VM: NTP, httpd for the installer artifacts, HAProxy (optional), and the identity services (IdM/Keycloak).
