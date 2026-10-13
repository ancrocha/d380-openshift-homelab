# DNS Server
The DNS server owns `lab.local` and the reverse zone of the OCP network, and forwards everything else upstream. OpenShift depends on DNS more than on anything else in this lab: the API, the ingress wildcard and every node name have to resolve, forwards and backwards.

## Prerequisites
- The bastion is configured and the VM creation is prepared, see [02 - bastion.md](02%20-%20bastion.md). This step creates the VM itself (cloned from the template, static IP 10.10.0.4, gateway 10.10.0.2).
- The bastion routes and NATs the OCP network, so this VM can reach the internet.
- `vars/secrets.yaml` holds the Red Hat login.

This VM resolves names against the upstream resolver (1.1.1.1). It cannot use itself while bind is being installed.

## Run
On the bastion, in `installation/ansible-playbooks`:

```bash
ansible-playbook -i inventory.yaml installation.yaml --tags dns --ask-vault-pass
```

What the playbook does:

1. Creates the VM on Proxmox (from the bastion, skipped if it already exists), starts it and waits until it answers over SSH.
2. Registers the VM with Red Hat, so `dnf` works.
3. Sets the hostname `dns.lab.local` and installs `bind`, `bind-utils` and `firewalld`.
4. Opens the `dns` service in firewalld.
5. Writes `/etc/named.conf` and both zone files from the templates in `templates/`, each validated (`named-checkconf`, `named-checkzone`) before it is put in place.
6. Enables and starts `named`, and reloads it when a file changed.
7. Points the bastion at the lab DNS: on its OCP connection it sets DNS `10.10.0.4`, search domain `lab.local` and DNS priority 50. That puts the lab DNS ahead of the home DNS in `/etc/resolv.conf`. The lab DNS forwards public names upstream, and the home DNS stays as a fallback if the DNS VM is down. The change is applied with `nmcli device reapply`, without taking the connection down. Because cloud-init configured the bastion's DNS, it told NetworkManager to leave `/etc/resolv.conf` alone (`dns = none` in `/etc/NetworkManager/conf.d/99-cloud-init.conf`); the playbook overrides that with `99-zz-lab-dns.conf` so NetworkManager writes the file again.

From then on the bastion reaches the lab VMs by name, e.g. `ssh registry.lab.local` (`cloud-user` and `~/.ssh/id_ed25519` are the defaults, so no `-i` is needed).

## The records
All of it comes from `vars/main.yaml`, so the zone files are generated and never edited by hand:

- `dns_records` - the lab infrastructure (bastion, dhcp, dns, registry)
- `dns_ocp_records` - the cluster: `api.ocp4`, `api-int.ocp4`, the `*.apps.ocp4` wildcard, the masters and the workers
- `dns_allow_query` - which networks may query (the OCP network and the home network)
- `lab_upstream_dns` - where public names are forwarded

The reverse zone is generated from the same two lists, one PTR per address, so forward and reverse can never drift apart.

**Bump `dns_zone_serial` whenever you change the records.** A zone whose serial does not move is a zone that caches stale answers.

## Verify
From the bastion or any node in the OCP network:
```bash
dig +short @10.10.0.4 registry.lab.local        # 10.10.0.5
dig +short @10.10.0.4 -x 10.10.0.5              # registry.lab.local.
dig +short @10.10.0.4 console.apps.ocp4.lab.local   # 10.10.0.11 (wildcard)
dig +short @10.10.0.4 redhat.com                # forwarded upstream
```

## From your PC
To reach the lab by name from your own machine, point `lab.local` at the DNS server and add a route to the OCP network:

- conditional forwarder for `lab.local` to 10.10.0.4 on your router or Pi-hole, or set your DNS to it
- static route: `route -p add 10.10.0.0 mask 255.255.255.0 10.0.0.50` (Windows), or add it on the home router so every device gets it

Next: [05 - registry.md](05%20-%20registry.md).
