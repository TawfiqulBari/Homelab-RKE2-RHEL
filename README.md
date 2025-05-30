# Homelab-RKE2-RHEL
Run a 5 node RKE2 based on RHEL - 8.7
```
systemctl disable apparmor.service
systemctl disable firewalld.service
systemctl stop apparmor.service
systemctl stop firewalld.service
```
```
subscription-manager release --set=8.4 ;
yum clean all;
subscription-manager release --show;
rm -rf /var/cache/dnf
```
```
systemctl disable swap.target
swapoff -a
```
```
yum update -y
```

