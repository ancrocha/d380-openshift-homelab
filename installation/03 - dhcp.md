# DHCP Server
The DHCP server hands out addresses on the isolated OCP network (10.10.0.0/24). The OpenShift nodes run RHCOS and have no cloud-init, so this is how they get their addressing.

The dhcp and dns VMs do not use it: cloud-init gives them a static address, because they are the services everything else depends on. Every VM after them (registry, the OpenShift nodes) has no `ip` in `vars/main.yaml` and gets its address, gateway and DNS from this server, through a fixed lease on its MAC.

## Prerequisites
- The bastion is configured and the VM creation is prepared (SSH key, `vars/secrets.yaml`), see [02 - bastion.md](02%20-%20bastion.md). This step creates the VM itself (cloned from the template, static IP 10.10.0.3, gateway 10.10.0.2).
- The bastion routes and NATs the OCP network, so this VM can reach the internet.
- `vars/secrets.yaml` holds the Red Hat login.

This VM resolves names against the upstream resolver (1.1.1.1), not against the lab DNS: it is built before the DNS server exists.

## Run
On the bastion, in `installation/ansible-playbooks`:

```bash
ansible-playbook -i inventory.yaml installation.yaml --tags dhcp --ask-vault-pass
```

What the playbook does:

1. Creates the VM on Proxmox (from the bastion, skipped if it already exists), starts it and waits until it answers over SSH.
2. Registers the VM with Red Hat, so `dnf` works.
3. Sets the hostname `dhcp.lab.local` and installs `dhcp-server` and `firewalld`.
4. Opens the `dhcp` service in firewalld.
5. Writes `/etc/dhcp/dhcpd.conf` from `templates/dhcpd.conf.j2` and validates it with `dhcpd -t` before putting it in place.
6. Enables and starts `dhcpd`, and restarts it when the configuration changed.

## What it serves
From `vars/main.yaml`:

- pool `10.10.0.200` to `10.10.0.250`
- default gateway `10.10.0.2` (the bastion)
- DNS server `10.10.0.4` and domain name `lab.local`
- DNS search domain `lab.local`
- fixed leases from `dhcp_reservations` (for now the registry: `BC:24:11:00:00:05` gets 10.10.0.5 and the host name `registry.lab.local`)

The OpenShift nodes need predictable addresses, so each one gets an entry in `dhcp_reservations` with the MAC from `proxmox_vms`. Their DNS records are in `dns_ocp_records` (see [04 - dns.md](04%20-%20dns.md)), and the two must agree.

## Verify
On the DHCP VM:
```bash
sudo systemctl status dhcpd
sudo journalctl -u dhcpd -n 20
```
Once a node asks for an address, the leases show up in `/var/lib/dhcpd/dhcpd.leases`.

Next: [04 - dns.md](04%20-%20dns.md).
