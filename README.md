# eBPF-OSC-Install
Creating an eBPF compatible cluster for Open-Source Calico Installation

The easiest way to start an EKS cluster that meets eBPF mode’s requirements is to use Amazon’s Bottlerocket OS, instead of the default. Bottlerocket is a container-optimised OS with an emphasis on security; it has a version of the kernel which is compatible with eBPF mode.

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

To create a 2-node test cluster with a Bottlerocket node group, run the command below. 
```
wget https://raw.githubusercontent.com/n1g3ld0uglas/eBPF-OSC-Install/main/cluster.config.yaml
```

It is important to use the config-file approach to creating a cluster in order to set the additional IAM permissions for Bottlerocket.
```
eksctl create cluster --config-file cluster.config.yaml
```


Install the Tigera Operator
```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
```

Download the installation file
```
wget https://raw.githubusercontent.com/n1g3ld0uglas/eBPF-OSC-Install/main/install.yaml
```

Using kubectl, apply the following Installation resource to tell the operator to install Calico; note the flexVolumePath tweak, which is needed for Bottlerocket.
```
kubectl apply -f install.yaml
```

Confirm all Calico pods are running
```
kubectl get pods -A
```

Confirm all Calico nodes are healthy
```
kubectl get nodes -A
```

# Configure Calico to connect directly to the API server
When configuring Calico to connect to the API server, we need to use the load balanced domain name created by EKS. 
It can be extracted from kube-proxy’s config map by running:

```
kubectl get cm -n kube-system kube-proxy -o yaml | grep server
```

which should show the server name, for example:
```
server: https://d881b853ae9313e00302a84f1e346a77.gr7.us-west-2.eks.amazonaws.com
```

In this example, you would use d881b853ae9313e00302a84f1e346a77.gr7.us-west-2.eks.amazonaws.com for KUBERNETES_SERVICE_HOST and 443 (the default for HTTPS) for KUBERNETES_SERVICE_PORT when creating the config map.

# Configuring the ConfigMap with server and port credentials

Since we used the operator to install Calico, create the following config map in the calico-system namespace using the host and port determined above:

```
wget config-map.yaml
```

```
vi config-map.yaml
```

```
kubectl apply -f config-map.yaml
```
