# Bastion Host. 
Procedures to install and prepare the bastion host.

## Create The VM

1. Find the specs in the architecture file.
2. Use the recommended specs:
```
CPU type: host
Machine: q35
Disk: enable discart and iothread
```

## OS Install

Install RHEL 9 (free: a Red Hat Developer subscription covers 16 systems)

### OS, routing, NTP, lab CA

```bash
# Find your NIC names first. On q35 they are enpXsY, not ensNN.
ip -br a
IF_HOME=enp6s18   # home bridge, 10.0.0.50/24, gw 10.0.0.1
IF_OCP=enp6s19    # OCP bridge,  10.10.0.2/24, no gateway

sudo hostnamectl set-hostname bastion.lab.local

# Look up the connection profile by device, because profile names vary
con() { nmcli -g GENERAL.CONNECTION dev show "$1"; }
sudo nmcli con mod "$(con $IF_OCP)"  ipv4.addresses 10.10.0.2/24 ipv4.method manual ipv4.dns "" connection.zone internal
sudo nmcli con mod "$(con $IF_HOME)" connection.zone external
sudo nmcli con up "$(con $IF_OCP)"; sudo nmcli con up "$(con $IF_HOME)"

# Verify before continuing
sudo firewall-cmd --get-active-zones
# expected: external -> home NIC, internal -> OCP NIC

# NAT and forwarding so the helper VMs can dnf/podman/subscribe during setup
sudo firewall-cmd --permanent --zone=external --add-masquerade
sudo firewall-cmd --permanent --zone=internal --add-service={dns,ntp,dhcp,http,https}
sudo firewall-cmd --permanent --zone=internal --add-port={6443/tcp,22623/tcp}   # only with the optional HAProxy (A.11)

# firewalld 1.x: explicit internal -> external forwarding policy
sudo firewall-cmd --permanent --new-policy int-to-ext
sudo firewall-cmd --permanent --policy int-to-ext --add-ingress-zone internal
sudo firewall-cmd --permanent --policy int-to-ext --add-egress-zone external
sudo firewall-cmd --permanent --policy int-to-ext --set-target ACCEPT
sudo firewall-cmd --permanent --policy int-to-ext --add-masquerade
sudo firewall-cmd --reload

echo net.ipv4.ip_forward=1 | sudo tee /etc/sysctl.d/90-fwd.conf
sudo sysctl --system

# Verify
sudo firewall-cmd --zone=internal --list-services
sudo firewall-cmd --info-policy=int-to-ext

sudo dnf install -y bind bind-utils chrony httpd podman jq git tar openssl openldap-clients nfs-utils policycoreutils-python-utils
```

### NTP
The agent installer refuses to start if clocks drift

```bash
sudo tee -a /etc/chrony.conf <<'EOF'
allow 10.10.0.0/24
local stratum 10
EOF
sudo systemctl enable --now chronyd
```

### Lab CA
One time key generation. Keep lab-ca.key safe.

```bash
mkdir -p ~/lab-ca && cd ~/lab-ca
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -keyout lab-ca.key -out lab-ca.pem -subj "/CN=Lab Root CA/O=lab.local" \
  -addext "basicConstraints=critical,CA:TRUE" -addext "keyUsage=critical,keyCertSign,cRLSign"

# Helper script to generate certificates
# usage: ./mkcert.sh registry.lab.local [extra.san ...]
cat > mkcert.sh <<'EOF'
#!/bin/bash
set -e; n=$1; shift; san="DNS:$n"; for s in "$@"; do san="$san,DNS:$s"; done
openssl req -newkey rsa:2048 -nodes -keyout $n.key -out $n.csr -subj "/CN=$n"
openssl x509 -req -in $n.csr -CA lab-ca.pem -CAkey lab-ca.key -CAcreateserial -days 825 -sha256 \
  -out $n.crt -extfile <(printf "subjectAltName=$san\nextendedKeyUsage=serverAuth")
cat $n.crt lab-ca.pem > $n.fullchain.crt
EOF

chmod +x mkcert.sh

# Generate Keys for each service.
./mkcert.sh registry.lab.local
# ./mkcert.sh keycloak.lab.local
# ./mkcert.sh git.lab.local
# ./mkcert.sh s3.lab.local
# ./mkcert.sh mail.lab.local
# ./mkcert.sh syslog.lab.local

# trust the CA on the bastion itself
sudo cp lab-ca.pem /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust

```

## DNS
The bastion Host will serve as DNS server.


Edit the file: `/etc/named.conf` — change the options block:

```bash
options {
    listen-on port 53 { 127.0.0.1; 10.10.0.2; 10.0.0.50; };
    directory "/var/named";
    allow-query { localhost; 10.10.0.0/24; 10.0.0.0/24; };
    recursion yes;
    forwarders { 1.1.1.1; 8.8.8.8; };   # for the helper VMs during setup
    dnssec-validation no;
};
zone "lab.local" IN { type master; file "lab.local.zone"; allow-update { none; }; };
zone "0.10.10.in-addr.arpa" IN { type master; file "0.10.10.rev"; allow-update { none; }; };
```

Create the file `/var/named/lab.local.zone`

```bash
$TTL 300
@   IN SOA bastion.lab.local. admin.lab.local. ( 2026100501 3600 900 604800 300 )
@   IN NS  bastion.lab.local.
bastion   IN A 10.10.0.2
registry  IN A 10.10.0.3
# nfs       IN A 10.10.0.4
# idm       IN A 10.10.0.10
# keycloak  IN A 10.10.0.11
# git       IN A 10.10.0.12
# ceph      IN A 10.10.0.13
# s3        IN A 10.10.0.14
# mail      IN A 10.10.0.15
# syslog    IN A 10.10.0.16
; --- cluster ocp4 ---
api.ocp4      IN A 10.10.0.5
api-int.ocp4  IN A 10.10.0.5
*.apps.ocp4   IN A 10.10.0.6
master01.ocp4 IN A 10.10.0.21
master02.ocp4 IN A 10.10.0.22
master03.ocp4 IN A 10.10.0.23
worker01.ocp4 IN A 10.10.0.31
worker02.ocp4 IN A 10.10.0.32
; short names too (the chapters use master01.lab.local)
master01 IN CNAME master01.ocp4
master02 IN CNAME master02.ocp4
master03 IN CNAME master03.ocp4
worker01 IN CNAME worker01.ocp4
worker02 IN CNAME worker02.ocp4
```

Create the file `/var/named/0.10.10.rev`

```bash
$TTL 300
@   IN SOA bastion.lab.local. admin.lab.local. ( 2026100501 3600 900 604800 300 )
@   IN NS  bastion.lab.local.
2   IN PTR bastion.lab.local.
3   IN PTR registry.lab.local.
# 4   IN PTR nfs.lab.local.
# 10  IN PTR idm.lab.local.
# 11  IN PTR keycloak.lab.local.
# 12  IN PTR git.lab.local.
# 13  IN PTR ceph.lab.local.
# 14  IN PTR s3.lab.local.
# 15  IN PTR mail.lab.local.
# 16  IN PTR syslog.lab.local.
5   IN PTR api.ocp4.lab.local.
21  IN PTR master01.ocp4.lab.local.
22  IN PTR master02.ocp4.lab.local.
23  IN PTR master03.ocp4.lab.local.
31  IN PTR worker01.ocp4.lab.local.
32  IN PTR worker02.ocp4.lab.local.
```

Apply the settings:
```bash
sudo chown root:named /var/named/lab.local.zone /var/named/0.10.10.rev
sudo named-checkconf && sudo named-checkzone lab.local /var/named/lab.local.zone
sudo systemctl enable --now named

sudo nmcli con mod "$(con $IF_OCP)"  ipv4.dns 127.0.0.1
sudo nmcli con mod "$(con $IF_HOME)" ipv4.ignore-auto-dns yes ipv4.dns 127.0.0.1
sudo nmcli con up "$(con $IF_HOME)"; sudo nmcli con up "$(con $IF_OCP)"
```

On your PC: set DNS to 10.0.0.50 (or add a conditional forwarder for lab.local → 10.0.0.50 on your router / Pi-hole), and add a static route 10.10.0.0/24 via 10.0.0.50 (Windows: route -p add 10.10.0.0 mask 255.255.255.0 10.0.0.50; or add it on the home router so every device gets it). Then https://console-openshift-console.apps.ocp4.lab.local works from your browser.

## DHCP
This will provide the IP addressing for all VMs in the cluster.

```bash
sudo dnf install dhcp-server -y
```

Configure `/etc/dhcp/dhcpd.conf`

```bash
# Global options
option domain-name "lab.local";
option domain-name-servers 10.10.0.2;
option ntp-servers 10.10.0.2;

default-lease-time 600;
max-lease-time 7200;

# Specify this DHCP server as authoritative for the local network
authoritative;

# Subnet configuration
subnet 10.10.0.0 netmask 255.255.255.0 {
    range 10.10.0.200 10.10.0.250;     # IP address allocation pool
    option routers 10.10.0.2;           # Default gateway
    option broadcast-address 10.10.0.255;
}

# Optional: IP Reservation (Static Lease) for a specific device
host registry {
    hardware ethernet 00:11:22:33:44:55; # Client MAC Address
    fixed-address 10.10.0.3;         # Assigned IP Address
    option host-name "registry.lab.local";
}
```

Restart the dhcp `sudo systemctl restart dhcpd`