# eBPF Open-Source Calico Installation
*Creating an eBPF compatible cluster for Open-Source Calico Installation*

The easiest way to start an EKS cluster that meets eBPF mode’s requirements is to use Amazon’s Bottlerocket OS, instead of the default. Bottlerocket is a container-optimised OS with an emphasis on security; it has a version of the kernel which is compatible with eBPF mode.

## Installing EKSCTL
Download and extract the latest release of eksctl with the following command
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```
Move the extracted binary to /usr/local/bin.
```
sudo mv /tmp/eksctl /usr/local/bin
```
Test that your installation was successful with the following command
```
eksctl version
```

## Installing KUBECTL
Download the vended kubectl binary for your cluster's Kubernetes version from Amazon S3
```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
```
Apply execute permissions to the binary
```
chmod +x ./kubectl
```
Create a $HOME/bin/kubectl and ensuring that $HOME/bin comes first in your $PATH
```
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
```
Add $HOME/bin path to your shell initialization file so it's configured when opening shell
```
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
```
After you install kubectl , you can verify its version with the following command
```
kubectl version --short --client
```

## Installing a 3 node cluster
To create a 3-node test cluster with a Bottlerocket node group, run the command below. 
```
wget https://raw.githubusercontent.com/n1g3ld0uglas/eBPF-OSC-Install/main/cluster.config.yaml
```

Make sure your cluster meets the needs of your project:
```
cat cluster.config.yaml
```

It is important to use the config-file approach to creating a cluster in order to set the additional IAM permissions for Bottlerocket.
```
eksctl create cluster --config-file cluster.config.yaml
```

Confirm the node cluster was correctly configured:
```
kubectl get nodes -A
```


## Installing the Tigera Operator
Install the Tigera Operator
```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
```

Download the installation file
```
wget https://raw.githubusercontent.com/n1g3ld0uglas/eBPF-OSC-Install/main/install.yaml
```

Confirm Calico pods are not running:
```
kubectl get pods -A
```

Using kubectl, apply the following Installation resource to tell the operator to install Calico; note the flexVolumePath tweak, which is needed for Bottlerocket.
```
kubectl apply -f install.yaml
```

Test again and confirm all Calico pods are running
```
kubectl get pods -A
```


## Configure Calico to connect directly to the API server
When configuring Calico to connect to the API server, we need to use the load balanced domain name created by EKS. 
It can be extracted from kube-proxy’s config map by running:

```
kubectl get cm -n kube-system kube-proxy -o yaml | grep server
```

which should show the server name, for example:
```
server: https://d881b853ae9313e00302a84f1e346a77.gr7.us-west-2.eks.amazonaws.com
```

## Configuring the ConfigMap with server and port credentials

Since we used the operator to install Calico, create the following config map in the calico-system namespace using the host and port determined above:
```
wget https://raw.githubusercontent.com/n1g3ld0uglas/eBPF-OSC-Install/main/configmap.yaml
```

In this example, you would use d881b853ae9313e00302a84f1e346a77.gr7.us-west-2.eks.amazonaws.com for KUBERNETES_SERVICE_HOST and 443 (the default for HTTPS) for KUBERNETES_SERVICE_PORT when creating the config map.
```
vi config-map.yaml
```

Apply changes to the updated file
```
kubectl apply -f config-map.yaml
```

Wait 60s for kubelet to pick up the ConfigMap (see Kubernetes issue #30189); then, restart the operator to pick up the change:
```
kubectl delete pod -n tigera-operator -l k8s-app=tigera-operator
```

## Disable Kube-Proxy

In eBPF mode, Calico replaces kube-proxy so it wastes resources to run both. To disable kube-proxy reversibly, we recommend adding a node selector to kube-proxy’s DaemonSet that matches no nodes, for example:
```
kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
```

Then, should you want to start kube-proxy again, you can simply remove the node selector.

## Enable eBPF Mode

To enable eBPF mode, change the spec.calicoNetwork.linuxDataplane parameter in the operator’s Installation resource to "BPF"; you must also clear the hostPorts setting because host ports are not supported in BPF mode:
```
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"BPF", "hostPorts":null}}}'
```

## Disabling eBPF Mode

Follow these steps if you want to switch from Calico’s eBPF dataplane back to standard Linux networking:

* Revert the changes to the operator’s installation resource:
```
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"Iptables"}}}'
```

* If you disabled kube-proxy, re-enable it (for example, by removing the node selector added above).
```
kubectl patch ds -n kube-system kube-proxy --type merge -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": null}}}}}'
```

* Since disabling eBPF mode is disruptive, monitor existing workloads to make sure they reestablish connections.


## Patch the Felix Agent:

If you choose not to disable kube-proxy (for example, because it is managed by your Kubernetes distribution), then you must change Felix configuration parameter BPFKubeProxyIptablesCleanupEnabled to false. 

Download the latest version of calicoctl
```
curl -o calicoctl -O -L  "https://github.com/projectcalico/calicoctl/releases/download/v3.19.1/calicoctl" 
```

Make it an executable program:
```
chmod +x calicoctl
```

Make the changes via calicoctl as follows:
```
./calicoctl patch felixconfiguration default --patch='{"spec": {"bpfKubeProxyIptablesCleanupEnabled": true}}'
```

Confirm changes were enforced:
```
./calicoctl get felixconfiguration default -o yaml
```
