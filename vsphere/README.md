## Prepare vCenter

Create datacenter e.g. "dc1"
Create a folder (e.g. "vmware-test") for "VM and Templates" in the datacenter
Import the ESXi host into vCenter


# Prepare environment variables and render templates

```
# Add token to avoid GitHub throttling  
export GITHUB_TOKEN=
. secrets/envsecrets.sh

# Select if cluster is migrated from existing or a new standard cluster
#export CLUSTER_TYPE=standard
export CLUSTER_TYPE=migration

kind create cluster -n kind-talos-vsphere-poc

. ../standard/${CLUSTER_TYPE}.env
kubectl create ns ${MGMT_NAMESPACE}
envsubst < ../standard/ipam.yaml >ipam.yaml
envsubst < ../standard/${CLUSTER_TYPE}.yaml >cluster.yaml
envsubst < ../standard/cpi-secrets.yaml >cpi-secrets.yaml

export KUBERNETES_API_SERVER_ADDRESS=${CONTROL_PLANE_ENDPOINT}
export KUBERNETES_API_SERVER_PORT=6443
```
## Prepare Cilium manifests

```
helm repo add cilium https://helm.cilium.io/
helm repo update

helm template cilium cilium/cilium \
    --version 1.11.2 \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=strict \
    --set k8sServiceHost="${KUBERNETES_API_SERVER_ADDRESS}" \
    --set k8sServicePort="${KUBERNETES_API_SERVER_PORT}" \
    --set enableXTSocketFallback=false \
    --set hubble.listenAddress=":4244" \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true > ../cni/cilium-v1.11.2.yaml

```
Push cilium to Git repo

# Bootstrap cluster using a local `KinD` cluster
```
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
kubectl apply -f ipam.yaml -n ${MGMT_NAMESPACE}
kubectl apply -f secrets/${CLUSTER_NAME}-secret.yaml -n ${MGMT_NAMESPACE}
kubectl apply -f cluster.yaml -n ${MGMT_NAMESPACE}

# Get config
export configname=`kubectl get talosconfig -n ${MGMT_NAMESPACE} | grep ${CLUSTER_NAME}-controlplane | cut -d' ' -f1 | head -1`
kubectl get talosconfig -n ${MGMT_NAMESPACE} ${configname}  -o yaml -o jsonpath='{.status.talosConfig}' > ${CLUSTER_NAME}-talosconfig

kubectl get secret -n ${MGMT_NAMESPACE} ${CLUSTER_NAME}-talosconfig -o jsonpath='{.data.talosconfig}' | base64 -d > ${CLUSTER_NAME}-cluster-talosconfig
#talosctl config merge --talosconfig ./${CLUSTER_NAME}-talosconfig ${CLUSTER_NAME}-cluster-talosconfig

# Work-a-round: Set IP address of control plane
export IP=172.16.42.230

talosctl -n ${IP} --endpoints ${IP} version

# Only for new clusters (not migrated clusters)
talosctl bootstrap --talosconfig ./${CLUSTER_NAME}-talosconfig --endpoints ${IP} --nodes ${IP}

# Switch kubectl to remote Talos cluster
export KUBECONFIG=./kubeconfig-remote
export IP=172.16.42.230
talosctl  -n ${IP} --talosconfig=./${CLUSTER_NAME}-talosconfig --endpoints ${IP} kubeconfig

# Add secret and configuration for CPI
kubectl apply -f cpi-secrets.yaml
kubectl apply -f secrets/cilium-secrets.yaml


talosctl ls --talosconfig ./${CLUSTER_NAME}-talosconfig --endpoints ${IP} --nodes ${IP}

# Verify nodes
kubectl get nodes


# Check cluster status
export KUBECONFIG=
clusterctl describe cluster  vmware-test -n vmware-test

# Check the kernel log from a node
talosctl --talosconfig ./${CLUSTER_NAME}-talosconfig --endpoints ${IP} --nodes ${IP} dmesg

```
...

# Delete cluster
kubectl delete -f cluster.yaml -n vmware-test


Note, if the `Kind` cluster is delete without deleting the content from Cluster.yaml then virtual machine must be deleted manually in vCenter.
```
kind delete cluster -n kind-talos-vsphere-poc

```

###############
#
# PoC - migration from Kubeadm cluster to Talos
#
#################
```
kind create cluster -n kind-kubeadm-vsphere-poc
kubectl create ns source
kubectl apply -f secrets/source-secret.yaml -n source
. secrets/envsecrets.sh
. ../kubeadm/kubeadm.env
envsubst < ../kubeadm/ipam.yaml >ipam.yaml
envsubst < ../kubeadm/cpi-secrets.yaml >cpi-secrets.yaml
envsubst < ../kubeadm/standard.yaml >cluster.yaml

export KUBERNETES_API_SERVER_ADDRESS=${CONTROL_PLANE_ENDPOINT}
export KUBERNETES_API_SERVER_PORT=6443

#clusterctl generate ${CLUSTER_NAME} \
#   --infrastructure vsphere \
#    --kubernetes-version ${KUBERNETES_VERSION} \
#    --control-plane-machine-count 3 \
#    --worker-machine-count 2 \
#    --from ../kubeadm/source.yaml > cluster.yaml

clusterctl init --bootstrap kubeadm --control-plane kubeadm --infrastructure vsphere
clusterctl init --ipam in-cluster

# See that pods are ready
kubectl get pods -A

# Create the cluster
kubectl apply -f ipam.yaml -n source
kubectl apply -f cluster.yaml -n source


kubectl get secret/source-kubeconfig -o json -n source | jq -r .data.value  | base64 --decode > ./source.kubeconfig

# For remote cluster:
export KUBECONFIG=source.kubeconfig
kubectl apply -f cpi-secrets.yaml
kubectl apply -f secrets/cilium-secrets.yaml
kubectl apply -f ../cni/cilium-v1.12.6.yaml

# Use Cilium until Calico fixes https://github.com/tigera/operator/issues/2444
#kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml


```


# Implementation notes

Had to:
* Import OVA into content library (got some IO errors, using "Deploy OVF" instead)
* Convert ova to VM using the same name for the vm and the snapshot - if imported OVA
* Adding disk to get som writeable storage. The disk should be added to the VM that is used for cloning
  ```
  govc vm.disk.change -vm vmware-amd64 -disk.name disk-1000-0 -size 10G
  ```
* take a snapshot  (c - did not take snapshot)
* convert back to template. 

Using VIP:
  Adding 2 additional control plane nodes resulted in broken connectivity to control-plane.
  The VIP is causing the problem. The same VIP is assigned multiple virtual machines. 
  kube-vip is replacing haproxy, maybe use BigIP DNS for giving a list of control plane IP addresses with alive check.

Looks like it is problematic to get etcd up an running again after other etcd instances have been removed.


Network pool:
https://github.com/spectrocloud/cluster-api-provider-vsphere-static-ip/blob/master/docs/workflow.md


the vSphere provider supports IPAM and pool of IP addresses,See also:
https://github.com/kubernetes-sigs/cluster-api/pull/6000



TODO:

* Add tolerations to Calico https://github.com/projectcalico/calico/issues/7070
* Setup timeout for waiting for pods to be drained


Build our own image of `ghcr.io/mologie/talos-vmtoolsd-unstable:latest` from
https://github.com/mologie/talos-vmtoolsd


What about this from vmtoolsd:
{"error":"rpc error: code = Unavailable desc = connection error: desc = \"transport: authentication handshake failed: x509: certificate signed by unknown authority (possibly because of \\\"x509: Ed25519 verification failure\\\" while trying to verify candidate authority certificate \\\"talos\\\")\"","level":"error","module":"talosapi","msg":"error retrieving hostname"}


Important notes about cloud-init and vSphere versions:
https://kb.vmware.com/s/article/90331


Note regarding vSphere versions:
https://github.com/siderolabs/talos/issues/3143



Work-a-rounds for local KinD cluster:
* Added local DNS server to core-dns as Docker subnet 192.168.65.2 does not respond to DNS lookups.


Rolling upgrade of control-plane nodes:
There must be something wrong with the count of control planes:
```
NAME                                                         READY  SEVERITY  REASON       SINCE  MESSAGE                                                                        
Cluster/vmware-test                                          False  Warning   ScalingDown  4s     Scaling down control plane to 824648306132 replicas (actual 4)                  
├─ClusterInfrastructure - VSphereCluster/vmware-test         True                          24m                                                                                    
├─ControlPlane - TalosControlPlane/vmware-test-controlplane  False  Warning   ScalingDown  4s     Scaling down control plane to 824648306132 replicas (actual 4)                  
│ └─4 Machines...                                            True                          21m    See vmware-test-controlplane-4w866, vmware-test-controlplane-jjx5f, ...         
└─Workers                                                                                                                                                                         
  └─MachineDeployment/vmware-test-workers                    True                          20m                                                                                    
    └─2 Machines...                                          True                          20m    See vmware-test-workers-6c87c7cf5d-8f9qb, vmware-test-workers-6c87c7cf5d-dd5td  
```

