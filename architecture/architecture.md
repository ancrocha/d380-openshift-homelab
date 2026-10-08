# Architecture
This is the proposed architecture of this lab. Note that multiple services could be groups in one single VM. This separation I performed is for learning purposes only.

## Domain
lab.local

## Nodes and functions

| Hostname | CPU | RAM | DISK | OS | IP | Purpose |
|----------|-----|-----|------|----|----|---------|
| bastion | 4 | 4 GB | 60 GB | RedHat 9 | 10.0.0.50 (home) <br> 10.10.0.2 (ocp) | Router/NAT, DNS, NTP, lab CA, httpd, HAProxy (optional), installer tools, jump host |
| registry | 2 | 4 GB | 300 GB | RedHat 9 | 10.10.0.3 | mirror registry (Quay) |

## Network connections

