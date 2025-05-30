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
yum -y install nano
```
Step 1: Prepare First Control Plane
1. Run the below commands to create required directories for RKE2 configurations.
```
mkdir -p /etc/rancher/rke2/
mkdir -p  /var/lib/rancher/rke2/server/manifests/
```
2. Create a deployment manifest called config.yaml for RKE2 Cluster and replace the IP addresses and corresponding FQDNS according.( add any other fields from the Extra Options sections in config.yaml  at this point )
```
cat<<EOF|tee /etc/rancher/rke2/config.yaml
tls-san:
  - homelab.tawfiqulbari.work
  - 192.168.0.40
  - rke2-control-1.tawfiqulbari.work
  - 192.168.0.41
  - rke2-control-2.tawfiqulbari.work
  - 192.168.0.42
  - rke2-control-3.tawfiqulbari.work
  - 192.168.0.43
write-kubeconfig-mode: "0600"
etcd-expose-metrics: true
cni:
  - canal

EOF
```
In above mentioned template manifest,

192.168.0.40 is the Kube-VIP  IP
homelab.tawfiqulbari.work is the Kube-VIP FQDN
remaining IPs and FQDN are for all 3 Control Planes

Step 2: Ingress-Nginx config for RKE2
1. By default RKE-2 based ingress controller doesn't allow additional snippet information in ingress manifests. Create this config before starting the deployment of RKE2.
```
cat<<EOF| tee /var/lib/rancher/rke2/server/manifests/rke2-ingress-nginx-config.yaml
---
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-ingress-nginx
  namespace: kube-system
spec:
  valuesContent: |-
    controller:
      metrics:
        service:
          annotations:
            prometheus.io/scrape: "true"
            prometheus.io/port: "10254"
      config:
        use-forwarded-headers: "true"
      allowSnippetAnnotations: "true"
EOF
```
2. Begin the RKE2 Deployment
```
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh -
```
```
systemctl start rke2-server
```
```
systemctl enable rke2-server
```
By default RKE2 deploys all the binaries in /var/lib/rancher/rke2/bin path. Add this path to system's default PATH for kubectl utility to work appropriately.
```
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
```
Also, append these lines into current user's .bashrc  file.
```
echo "export PATH=$PATH:/var/lib/rancher/rke2/bin" >> $HOME/.bashrc
echo "export KUBECONFIG=/etc/rancher/rke2/rke2.yaml"  >> $HOME/.bashrc
```
Get the token for joining other Control-Plane Nodes.
```
cat /var/lib/rancher/rke2/server/node-token
```
