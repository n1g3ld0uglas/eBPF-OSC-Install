# eBPF-OSC-Install
Creating an eBPF compatible cluster for Open-Source Calico Installation

To create a 2-node test cluster with a Bottlerocket node group, run the command below. 
```
kubectl create -f https://docs.projectcalico.org/manifests/clusterconfig.yaml
```


It is important to use the config-file approach to creating a cluster in order to set the additional IAM permissions for Bottlerocket.


Install the Tigera Operator
```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
```
