bootstrap cluster using a local `Kind` cluster

```
# Add token to avoid GitHub throttling  
export GITHUB_TOKEN=

kind create cluster -n kind-talos-vsphere-poc
k create ns vmware-test
k apply -f secrets/secret.yaml -n vmware-test
. ../standard/standard.env
envsubst < ../standard/standard.yaml >cluster.yaml

clusterctl init --bootstrap talos --control-plane talos --infrastructure vsphere

# Have to init kubeadm to avoid vSphere errors. Kubeadm controller is not used.
clusterctl init -c kubeadm

# See that pods are ready
k get pods -A

# Create the cluster
k apply -f cluster.yaml -n vmware-test

# For each vm attach a writable disk
govc vm.disk.change -vm vmware-test-controlplane-nw22r -disk.name disk-1000-0 -size 10G

# Power off vm to get Talos to pick up the new disk. The controller will power it on again.


# Get config
k get talosconfig -n vmware-test vmware-test-controlplane-${ID}  -o yaml -o jsonpath='{.status.talosConfig}' > talosconfig


talosctl bootstrap --talosconfig ./talosconfig --endpoints ${IP} --nodes ${IP}

export KUBECONFIG=./kubeconfig-remote
talosctl  -n ${IP} --talosconfig=./talosconfig --endpoints ${IP} kubeconfig


```
...

Note, if the `Kind` cluster is delete without deleting the content from Cluster.yaml then virtual machine must be deleted manually in vCenter.
```
kind delete cluster -n kind-talos-vsphere-poc

```




Had to:
* Convert ova to VM, take a snapshot, convert back to template. Using the same name for the vm and the snapshot.

* Adding disk to get som writeable storage. How to get that into the Cluster API?
```
govc vm.disk.change -vm vmware-test-controlplane-ltr2c -disk.name disk-1000-0 -size 10G
```


Network:
How to get the IP address of the new VM? and how to send the traffic to the API service?

kube-vip is replacing haproxy, maybe use BigIP DNS for giving a list of control plane IP addresses with alive check.

Network pool:
https://github.com/spectrocloud/cluster-api-provider-vsphere-static-ip/blob/master/docs/workflow.md


Install vmwtool as DaemonSet, how do we get this onto the Control Plane nodes if API server is not running?
https://github.com/mologie/talos-vmtoolsd

