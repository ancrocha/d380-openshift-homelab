# Proxmox preparation
Everything the Ansible playbooks need on the Proxmox server before they can create VMs. Do this once, on the Proxmox host (UI or shell), before starting the bastion.

## API token
The playbooks talk to the Proxmox API with a token.

1. In the Proxmox UI go to Datacenter > Permissions > API Tokens > Add.
2. Use user `root@pam` and token ID `ansible`.
3. Untick "Privilege Separation" (lab only).
4. Copy the secret, it is shown only once. You will paste it into `vars/secrets.yaml` on the bastion.

## RHEL 9 template
The RHEL helper VMs (registry, ...) are cloned from this template and configured with cloud-init, so no OS install is needed.

Download the "Red Hat Enterprise Linux 9 KVM Guest Image" (`.qcow2`) from the Red Hat Customer Portal. The Proxmox UI upload only accepts `.iso`/`.img`, so rename the file to `rhel-9.8-x86_64-kvm.img` (the content stays qcow2) and upload it in Storage `local` > ISO Images > Upload. It is stored in `/var/lib/vz/template/iso/`.

Alternatively copy it with `scp` from PowerShell (adjust the file name and the Proxmox IP) and use `/root/rhel-9.8-x86_64-kvm.qcow2` as the path below:

```powershell
scp "$HOME\Downloads\rhel-9.8-x86_64-kvm.qcow2" root@10.0.0.6:/root/rhel-9.8-x86_64-kvm.qcow2
```

Then run as root on the Proxmox shell. Adjust the storage name (`local-lvm`) and the file path (add `--format qcow2` to the import if it complains about the format):

```bash
qm create 9000 --name rhel9-template --memory 2048 --cores 2 --cpu host --machine q35 \
  --scsihw virtio-scsi-single --net0 virtio,bridge=vmbr0 --agent enabled=1 --ostype l26
qm disk import 9000 /var/lib/vz/template/iso/rhel-9.8-x86_64-kvm.img local-lvm
qm set 9000 --scsi0 local-lvm:vm-9000-disk-0,discard=on,iothread=1
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot order=scsi0 --serial0 socket --vga serial0
qm set 9000 --ciupgrade 0    # no "dnf upgrade" on first boot: VMs are not registered yet, Ansible handles packages
qm template 9000
```

The template name (`rhel9-template`) and the disk (`scsi0`) must match `template` and `proxmox_template_disk` in `installation/ansible-playbooks/vars/main.yaml`.

## DHCP network snippet
Most lab VMs get their address, gateway and DNS from the lab DHCP server. Proxmox's own cloud-init network config always includes a DNS server (the Proxmox host's, when none is set), and cloud-init writes it into the VM over what DHCP hands out. So those VMs use a custom network config that only says "DHCP".

1. Allow snippets on the `local` storage: Datacenter > Storage > `local` > Edit > Content, add **Snippets** (keep the types already selected).
2. Create the snippet on the Proxmox shell:

```bash
cat > /var/lib/vz/snippets/dhcp-network.yaml <<'EOF'
# cloud-init network config: DHCP only, no DNS, so DNS comes from the DHCP lease
version: 2
ethernets:
  eth0:
    dhcp4: true
EOF
```

The playbook attaches it to every VM without an `ip` in `proxmox_vms` (`proxmox_dhcp_network_snippet` in `vars/main.yaml`). The user and the SSH key still come from the normal Proxmox cloud-init settings.

## Bastion VM
The bastion is the only VM created by hand, because the playbooks that create the other VMs run on it. It is cloned from the template above. Specs come from `architecture/architecture.md` (4 CPU, 4 GB RAM, 60 GB disk), with two NICs: `vmbr0` (home network, 10.0.0.50) and `ocp` (isolated OCP network).

On the Proxmox host, save the SSH public key of your PC (the one you will use to log in) to `/root/pc.pub`, then run as root on the Proxmox shell. Adjust the storage name, the home gateway and DNS:

```bash
qm clone 9000 800 --name bastion --full --storage local-lvm
qm set 800 --cores 4 --memory 4096 --onboot 1
qm set 800 --net0 virtio,bridge=vmbr0 --net1 virtio,bridge=ocp
qm set 800 --ciuser cloud-user --sshkeys /root/pc.pub \
  --ipconfig0 ip=10.0.0.50/24,gw=10.0.0.1 --nameserver 1.1.1.1
qm resize 800 scsi0 60G
qm start 800
```

Only the home NIC (`net0`) is configured by cloud-init. The OCP NIC (`net1`) is configured later by the bastion playbook. When the VM is up, log in with `ssh cloud-user@10.0.0.50` and continue with [02 - bastion.md](02%20-%20bastion.md). RHEL registration is not done by cloud-init, so the bastion is registered by hand after the first login (see the guide); the other VMs are registered by the playbook.

## ISO images (OpenShift nodes only)
The OpenShift nodes run RHCOS, which has no cloud-init. Their VMs are created empty and booted from an ISO. Upload it in Storage > ISO Images and set it as `iso` for the VM in `vars/main.yaml`.
