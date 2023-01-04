bootstrap cluster using a local `Kind` cluster

```
# Add token to avoid GitHub throttling  
export GITHUB_TOKEN=
. secrets/envsecrets.sh

kind create cluster -n kind-talos-vsphere-poc
kubectl create ns vmware-test
kubectl apply -f secrets/secret.yaml -n vmware-test
. ../standard/standard.env
envsubst < ../standard/ipam.yaml >ipam.yaml
envsubst < ../standard/standard.yaml >cluster.yaml

clusterctl init --bootstrap talos --control-plane talos --infrastructure vsphere

# Have to init kubeadm to avoid vSphere errors. Kubeadm controller is not used.
clusterctl init -c kubeadm

# Add IPAM
# Update ~/.cluster-api/clusterctl.yaml as described here:
# https://github.com/telekom/cluster-api-ipam-provider-in-cluster/releases
clusterctl init --ipam in-cluster


# See that pods are ready
kubectl get pods -A

# Create the cluster
kubectl apply -f ipam.yaml -n vmware-test
kubectl apply -f cluster.yaml -n vmware-test

# Not needed if disk is added to vm that is cloned from
# For each vm attach a writable disk
# govc vm.disk.change -vm vmware-test-controlplane-${ID} -disk.name disk-1000-0 -size 10G
# Power off vm to get Talos to pick up the new disk. The controller will power it on again.

# Get config
export configname=`kubectl get talosconfig -n vmware-test | grep vmware-test-controlplane | cut -d' ' -f1`
kubectl get talosconfig -n vmware-test ${configname}  -o yaml -o jsonpath='{.status.talosConfig}' > talosconfig

kubectl get secret --namespace vmware-test vmware-test-talosconfig -o jsonpath='{.data.talosconfig}' | base64 -d > cluster-talosconfig
talosctl config merge cluster-talosconfig
talosctl -n ${IP} version

talosctl bootstrap --talosconfig ./talosconfig --endpoints ${IP} --nodes ${IP}  --talosconfig=./talosconfig 

# Switch kubectl to remote Talos cluster
export KUBECONFIG=./kubeconfig-remote
talosctl  -n ${IP} --talosconfig=./talosconfig --endpoints ${IP} kubeconfig

# Install vmwtool as DaemonSet to let vSphere Center see the VM detailes like IP address
rm -f vmtoolsd-secret.yaml 
talosctl -n ${IP} --endpoints ${IP} config new vmtoolsd-secret.yaml --roles os:admin --talosconfig ./talosconfig
kubectl -n kube-system create secret generic talos-vmtoolsd-config   --from-file=talosconfig=./vmtoolsd-secret.yaml
kubectl apply -f ../standard/vmtools.yaml

# Must set taint before installing vSphere CPI driver.
# TODO: add to Talos configuration
kubectl taint nodes -l kubernetes.io/os=linux node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule

# Install CPI to set node.Spec.ProviderID. 
kubectl apply -f cpi-vsphere.yaml 

# Verify nodes
kubectl get nodes

# Check cluster status
export KUBECONFIG=
clusterctl describe cluster  vmware-test -n vmware-test


```
...

# Delete cluster
kubectl delete -f cluster.yaml -n vmware-test


Note, if the `Kind` cluster is delete without deleting the content from Cluster.yaml then virtual machine must be deleted manually in vCenter.
```
kind delete cluster -n kind-talos-vsphere-poc

```




Had to:
* Convert ova to VM, take a snapshot, convert back to template. Using the same name for the vm and the snapshot.
* Adding disk to get som writeable storage. The disk should be added to the VM that is used for cloning

Network:
How to get the IP address of the new VM? and how to send the traffic to the API service?

Using VIP:
Adding 2 additional control plane nodes resulted in broken connectivity to 192.168.0.11
The VIP is causing the problem. The same VIP is assigned multiple virtual machines. 
Looks like it is problematic to get etcd up an running again after other etcd instances have been removed.



kube-vip is replacing haproxy, maybe use BigIP DNS for giving a list of control plane IP addresses with alive check.

Network pool:
https://github.com/spectrocloud/cluster-api-provider-vsphere-static-ip/blob/master/docs/workflow.md


the vSphere provider supports IPAM and pool of IP addresses, however, the ClusterAPI does not yet support this. There is a proposal for this:
https://github.com/kubernetes-sigs/cluster-api/pull/6000




TODO:
Build our own image of `ghcr.io/mologie/talos-vmtoolsd-unstable:latest` from
https://github.com/mologie/talos-vmtoolsd


What about this from vmtoolsd:
{"error":"rpc error: code = Unavailable desc = connection error: desc = \"transport: authentication handshake failed: x509: certificate signed by unknown authority (possibly because of \\\"x509: Ed25519 verification failure\\\" while trying to verify candidate authority certificate \\\"talos\\\")\"","level":"error","module":"talosapi","msg":"error retrieving hostname"}


Important notes about cloud-init and vSphere versions:
https://kb.vmware.com/s/article/90331


Note regarding vSphere versions:
https://github.com/siderolabs/talos/issues/3143



Work-a-rounds:
* Added local DNS server to core-dns as Docker subnet 192.168.65.2 does not respond to DNS lookups.
* 

FAQ:

Delete a vm by deleting both vspheremachine and machine.