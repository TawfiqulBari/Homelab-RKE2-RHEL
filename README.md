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
Step 4: Deploy Kube-VIP
1. Decide the IP and the interface on all nodes for Kube-VIP and setup these as environment variables. This step must be completed before deploying any other node in the cluster (both CP and Workers).
```
export VIP=192.168.0.40
export INTERFACE=ens18
```
2. Import the RBAC manifest for Kube-VIP
```
curl https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/rke2/server/manifests/kube-vip-rbac.yaml
```
3. Fetch the kube-vip image
```
/var/lib/rancher/rke2/bin/crictl -r "unix:///run/k3s/containerd/containerd.sock"  pull ghcr.io/kube-vip/kube-vip:latest
```
4. Deploy the Kube-VIP
```
CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock  ctr -n k8s.io run \
--rm \
--net-host \
ghcr.io/kube-vip/kube-vip:latest vip /kube-vip manifest daemonset --arp --interface $INTERFACE --address $VIP --controlplane  --leaderElection --taint --services --inCluster | tee /var/lib/rancher/rke2/server/manifests/kube-vip.yaml
```
5. Wait for the kube-vip to complete bootstrapping
```
kubectl rollout status daemonset   kube-vip-ds    -n kube-system   --timeout=650s
```
6. Once the condition is met, you can check the daemonset by kube-vip is running 1 pod
```
kubectl  get ds -n kube-system  kube-vip-ds
```
Step 5: Remaining Control-Plane Nodes
Perform these steps on remaining control-plane nodes.

1. Create required directories for RKE2 configurations.
```
mkdir -p /etc/rancher/rke2/
mkdir -p  /var/lib/rancher/rke2/server/manifests/
```
2. Create a deployment manifest called config.yaml  for RKE2 Cluster  and replace the IP addresses and corresponding FQDNS according (add any other fields from the Extra Options sections in config.yaml  at this point).
```
cat<<EOF|tee /etc/rancher/rke2/config.yaml
server: https://192.168.0.40:9345
token: [token from /var/lib/rancher/rke2/server/node-token on server node 1]
write-kubeconfig-mode: "0644" tls-san:
  - homelab.tawfiqulbari.work
  - 192.168.0.40
  - rke2-control-1.tawfiqulbari.work
  - 192.168.0.41
  - rke2-control-2.tawfiqulbari.work
  - 192.168.0.42
  - rke2-control-3.tawfiqulbari.work
  - 192.168.0.43
write-kubeconfig-mode: "0644"
etcd-expose-metrics: true
cni:
  - canal

EOF
```
Step 6: Begin the RKE2 Deployment
1. Begin the RKE2 Deployment
```
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh -
```
2. Start the RKE2 service. Starting the Service will take approx. 10-15 minutes based on the network connection
```
systemctl start rke2-server
```
Enable the RKE2 Service
```
systemctl enable rke2-server
```
4. By default, RKE2 deploys all the binaries in /var/lib/rancher/rke2/bin  path. Add this path to system's default PATH for kubectl utility to work appropriately.
```
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
```
5. Also, append these lines into current user's .bashrc  file
```
echo "export PATH=$PATH:/var/lib/rancher/rke2/bin" >> $HOME/.bashrc
echo "export KUBECONFIG=/etc/rancher/rke2/rke2.yaml"  >> $HOME/.bashrc
```

