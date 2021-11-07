# EKS Distro (AWS base Kubernetes)
- https://snapcraft.io/eks

You might use the Vagrantfile attached. It will deploy the following Virtual Machines.
- One master node (No GPU machine)
- Two worker nodes (GPU machine and CPU machine)

|  | CPU | Memory | GPU | GPU Driver |
| --- | --- | --- | --- | --- |
| Master | 2 | 8,192 MB | no | --- |
| Worker1 | 2 | 16,384 MB | 1 | Not Installed |
| Worker2 | 2 | 8,192 MB | no | --- |

# 1. Install Snapd on Master/Worker nodes
```
$ sudo apt-get update
$ sudo apt-get install -y snapd
```

# 2. Install EKS thru Snap on Master/Worker nodes
```
$ sudo snap install eks --classic --edge
$ snap list
$ sudo eks status
$ snap services eks
```

# 3. Create the Cluster
First you should do is get the command which you run at each worker node.
```
$ sudo eks add-node
From the node you wish to join to this cluster, run the following:
eks join 192.168.121.253:25000/98fa6a5e4e30785d54eb861f25653180

If the node you are adding is not reachable through the default interface you can use one of the following:
 eks join 192.168.121.253:25000/98fa6a5e4e30785d54eb861f25653180
 eks join 192.168.33.120:25000/98fa6a5e4e30785d54eb861f25653180
 eks join 172.17.0.1:25000/98fa6a5e4e30785d54eb861f25653180
 eks join 10.1.219.64:25000/98fa6a5e4e30785d54eb861f25653180
```

# 4. Run eks join command at each Worker node
```
$ eks join 192.168.33.120:25000/20221b4e8f73f0fe9469a5eac3d728fb
Contacting cluster at 192.168.33.120
Waiting for this node to finish joining the cluster. ..
```

# 5. Check the Cluster on Master node
```
$ eks status
eks is running
high-availability: no
  datastore master nodes: 192.168.33.120:19001
  datastore standby nodes: none

$ ssh vagrant@192.168.33.121 sudo eks status
eks is running
high-availability: no
  datastore master nodes: 192.168.33.120:19001
  datastore standby nodes: none

$ eks kubectl get nodes -o wide
NAME      STATUS   ROLES    AGE   VERSION              INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
worker1   Ready    <none>   11m   v1.18.9-eks-1-18-1   192.168.121.145   <none>        Ubuntu 20.04.3 LTS   5.4.0-81-generic   containerd://1.3.7
master    Ready    <none>   28m   v1.18.9-eks-1-18-1   192.168.121.253   <none>        Ubuntu 20.04.3 LTS   5.4.0-81-generic   containerd://1.3.7

$ eks kubectl get pods -A -o wide
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE   IP                NODE      NOMINATED NODE   READINESS GATES
kube-system   aws-iam-authenticator-wkkqn                1/1     Running   1          31m   192.168.121.253   master    <none>           <none>
kube-system   calico-kube-controllers-555fc8cc5c-f7n5j   1/1     Running   0          32m   10.1.219.69       master    <none>           <none>
kube-system   hostpath-provisioner-66667bf7f-d2x7c       1/1     Running   0          32m   10.1.219.70       master    <none>           <none>
kube-system   metrics-server-768748c8f4-k47xf            1/1     Running   0          32m   10.1.219.71       master    <none>           <none>
kube-system   coredns-6788f546c9-bmbpv                   1/1     Running   0          32m   10.1.219.72       master    <none>           <none>
kube-system   calico-node-7bb5b                          1/1     Running   1          16m   192.168.121.253   master    <none>           <none>
kube-system   aws-iam-authenticator-fjzlp                1/1     Running   1          15m   192.168.121.145   worker1   <none>           <none>
kube-system   calico-node-ckd67                          1/1     Running   1          15m   192.168.121.145   worker1   <none>           <none>
```

# 6. Install Helm
```
$ eks kubectl config view --raw > $HOME/.kube/config
$ chmod 600 ~/.kube/config

$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 \
&& chmod 700 get_helm.sh \
&& ./get_helm.sh
```

# 7. Install GPU Operator
```
$ helm repo add nvidia https://nvidia.github.io/gpu-operator \
&& helm repo update

$ helm install --wait --generate-name \
nvidia/gpu-operator
```
