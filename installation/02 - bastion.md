# Bastion Host. 
The bastion host will have lots of functions in this cluster but the main one is to server as Ansible to provision the infrastructure.

## Prerequisite
The bastion VM is cloned from the RHEL 9 template and started on the Proxmox host, see [01 - proxmox.md](01%20-%20proxmox.md). It has two NICs (home and OCP) and cloud-init has configured the home NIC (10.0.0.50), so no OS install is needed.

## Server Configuration
Connect to the server using ssh (`ssh cloud-user@10.0.0.50`) and perform the following steps.

The bastion has to be registered with Red Hat by hand first, because the repositories are needed to install Ansible:

```bash
sudo subscription-manager register --username <redhat-login>
```

Install Ansible and get this repository:

```bash
sudo dnf update
sudo dnf install ansible-core -y

# Clone this repository
git clone https://github.com/ancrocha/d380-openshift-homelab.git
cd d380-openshift-homelab/installation/ansible-playbooks

# Download the requirements
ansible-galaxy collection install -r collections/requirements.yml

```

Run the bastion configuration (this also installs the Python libraries the Proxmox module needs, `proxmoxer` and `requests`):
```bash
ansible-playbook -i inventory.yaml installation.yaml --tags bastion

#reboot the machine to make hostname settings into effect.
sudo reboot

```

## Internet access for the OCP network
The OCP network (10.10.0.0/24) is isolated, so the bastion gives all the nodes in it internet access by routing and NAT through its home NIC. The bastion playbook configures this:

- the home NIC goes into the firewalld `external` zone and the OCP NIC into `internal`
- masquerade (NAT) is enabled on `external`
- a firewalld policy `int-to-ext` forwards traffic from `internal` to `external` (firewalld 1.x does not forward between zones without a policy)
- IPv4 forwarding is enabled (`/etc/sysctl.d/90-fwd.conf`)

Nodes in the OCP network must use the bastion (10.10.0.2) as their default gateway. The registry VM gets this from cloud-init.

After the playbook has run, verify on the bastion:
```bash
sudo firewall-cmd --get-active-zones            # external -> home NIC, internal -> OCP NIC
sudo firewall-cmd --info-policy=int-to-ext
sysctl net.ipv4.ip_forward                      # = 1
```
Then from a node in the OCP network (for example `ssh cloud-user@10.10.0.3`): `ping -c2 1.1.1.1` and `curl -sI https://access.redhat.com`. DNS names will only resolve once the bastion DNS is configured, or if the node uses a public resolver for now.

## Lab CA and certificates
One self-signed CA signs every certificate in the lab, so each node only has to trust one file (`lab-ca.pem`). The bastion playbook:

- creates the CA key and certificate in `~/lab-ca` (valid 10 years, created once, never overwritten)
- generates a key and a certificate for each name in `lab_certs` (`vars/main.yaml`, currently `registry.lab.local`). Each certificate has a subjectAltName and is valid 825 days. `<name>.fullchain.crt` is the certificate followed by the CA.
- trusts the CA on the bastion itself

To add a certificate for another service, add its name to `lab_certs` and run the bastion playbook again. Keep `~/lab-ca/lab-ca.key` safe: anyone with it can sign certificates your nodes trust.

Verify on the bastion:
```bash
ls ~/lab-ca
openssl verify -CAfile ~/lab-ca/lab-ca.pem ~/lab-ca/registry.lab.local.crt
```

## Prepare the creation of the other VMs
The playbook creates the remaining lab VMs through the Proxmox API. The VMs are defined in `vars/main.yaml` under `proxmox_vms`.

Each VM is created by its own step (03 dhcp, 04 dns, 05 registry), not all at once: the registry must only power on once DHCP and DNS are up. Here you only prepare what those steps need.

There are two kinds of VMs:
- **RHEL helper VMs** (registry, ...) are cloned from a RHEL 9 template and configured with cloud-init (user, SSH key, static IP). No OS install is needed.
- **OpenShift nodes** (RHCOS) have no cloud-init. They are created empty and booted from an ISO (`iso` in `vars/main.yaml`).

The API token, the RHEL template and the ISO images are prepared on the Proxmox server first, see [01 - proxmox.md](01%20-%20proxmox.md).

### 1. Playbook configuration
Edit `vars/main.yaml` and change the values marked `CHANGE`: Proxmox IP, node name and, for ISO based VMs, the ISO path. Each VM in `proxmox_vms` can set its own `storage` (any Proxmox storage name, e.g. one of the SSDs or the NVMe), otherwise `proxmox_vm_storage` is used. ISO based VMs can set their own `iso`, otherwise `proxmox_iso` is used.

Create the SSH key that cloud-init injects into the VMs (skip if you already have `~/.ssh/id_ed25519`):
```bash
ssh-keygen -t ed25519
```

Create the encrypted secrets file:
```bash
cp vars/secrets.example.yaml vars/secrets.yaml
vi vars/secrets.yaml                  # paste the API token secret from 01 - proxmox.md and your Red Hat login/password
ansible-vault encrypt vars/secrets.yaml
```

Next: [03 - dhcp.md](03%20-%20dhcp.md).

