# Registry Host
The registry host runs the mirror registry (Quay) that holds the OpenShift images for the disconnected installation. It is configured over SSH by the playbook, run from the bastion.

## Prerequisites
- DHCP ([03 - dhcp.md](03%20-%20dhcp.md)) and DNS ([04 - dns.md](04%20-%20dns.md)) are up. This step creates the VM itself (cloned from the template), so it only powers on now. It has no static IP: cloud-init only sets the user and the SSH key, and the VM gets 10.10.0.5, the gateway and the DNS server from its fixed DHCP lease (MAC `BC:24:11:00:00:05`).
- The DNS server resolves `registry.lab.local` and forwards public names, see [04 - dns.md](04%20-%20dns.md). DHCP hands out 10.10.0.4 as the resolver, so DNS has to work before the installer can be downloaded.
- The bastion has the lab CA and the `registry.lab.local` certificate in `~/lab-ca`.
- `vars/secrets.yaml` holds the Red Hat login and `quay_init_password`.

## Run
On the bastion, in `installation/ansible-playbooks`:

```bash
ansible-playbook -i inventory.yaml installation.yaml --tags registry --ask-vault-pass
```

What the playbook does, in order:

1. Creates the VM on Proxmox (from the bastion, skipped if it already exists), starts it and waits until it answers over SSH.
2. Registers the VM with Red Hat, so `dnf` works.
3. Sets the hostname `registry.lab.local` and installs `podman`, `polkit`, `firewalld` and `tar`.
4. Opens port 8443/tcp in firewalld.
5. Enables lingering for `cloud-user` (with a polkit rule that allows it), because the installer runs Quay as systemd user services.
6. Creates `/opt/quay`, copies `registry.lab.local.fullchain.crt` and `registry.lab.local.key` from the bastion's `~/lab-ca`, and trusts the lab CA.
7. Downloads the mirror registry installer from `mirror.openshift.com` and runs `mirror-registry install` with that certificate, `/opt/quay` as the root and the init user from `vars/main.yaml`. This takes several minutes. It is skipped if `/opt/quay/quay-config` already exists.

## Monitor the installation.

On the registry node:
```bash
podman ps -a
podman logs -f <container-id>        # the running container, usually the ansible-runner / installer one
podman images                        # quay, redis, postgres images appear as they load
```

## Verify
From the bastion:
```bash
curl -sI https://registry.lab.local:8443/health/instance
```
Log in to the web console with the init user and `quay_init_password`: `https://registry.lab.local:8443`. Your PC needs the route and the DNS forwarder from [04 - dns.md](04%20-%20dns.md), and you will see a certificate warning until you trust `lab-ca.pem` there.
