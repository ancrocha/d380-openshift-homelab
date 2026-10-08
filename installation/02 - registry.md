# Refistry Host. 
Procedures to install and prepare the Registry host.

## Create The VM

1. Find the specs in the architecture file.
2. Use the recommended specs:
```
CPU type: host
Machine: q35
Disk: enable discart and iothread
```

**Copy the MAC address and add to Bastion DHCP settings.**

## OS Install

Install RHEL 9 (free: a Red Hat Developer subscription covers 16 systems)

**Check you got the correct IPs from DHCP**

## Install Registry

**On bastion Host** Copy the KEY
```bash
ssh-keygen
ssh-copy-id andre@registry.lab.local

cd ~/lab-ca
scp registry.lab.local.fullchain.crt registry.lab.local.key lab-ca.pem andre@registry.lab.local:~/
```

**Back to registry**

```bash
sudo mkdir -p /opt/quay
sudo chown -R andre:andre /opt/quay
sudo firewall-cmd --permanent --add-port=8443/tcp && sudo firewall-cmd --reload

# Adding my user in sudo
echo 'andre ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/andre

#Instal Polkit
sudo dnf install -y polkit
sudo mkdir -p /etc/polkit-1/rules.d
sudo systemctl enable --now polkit

sudo tee /etc/polkit-1/rules.d/50-enable-linger.rules >/dev/null <<'EOF'
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.login1.set-user-linger" &&
        subject.user == "andre") {
        return polkit.Result.YES;
    }
});
EOF


#Install dependencies.
sudo loginctl enable-linger andre
sudo dnf install podman -y
```

```bash

# Download registry image
curl -LO https://mirror.openshift.com/pub/cgw/mirror-registry/latest/mirror-registry-amd64.tar.gz

# extract the mirror registry
mkdir -p ~/mirror-registry && tar -C ~/mirror-registry -xzf mirror-registry-amd64.tar.gz && cd ~/mirror-registry

./mirror-registry install -v \
  --quayHostname registry.lab.local \
  --quayRoot /opt/quay --quayStorage /opt/quay/storage \
  --initUser init --initPassword 'Redhat123!' \
  --sslCert ~/registry.lab.local.fullchain.crt \
  --sslKey  ~/registry.lab.local.key
```

You can access the node:

Add the bastion as a proxy host.
```
user: init
Pass: Redhat123!
https://registry.lab.local:8443/repository/
```