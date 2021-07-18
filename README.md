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
