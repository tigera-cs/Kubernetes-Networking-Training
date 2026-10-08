# Kubernetes Networking Training

This is the hands-on lab guide for Calico Networking Training.

## Lab setup

The following diagram depicts the lab setup used for the entire training.

<a href="img/lab-topology.png"><img src="img/lab-topology.png" alt="Lab topology" width="75%"></a>

Followings is a short description of each node in the lab.

* **Control1** is Kubernetes master node and runs Kubernetes control plane pods.
* **Worker1** is a Kubernetes worker node.
* **Worker2** is a Kubernetes worker node.
* **Bastion** is a node outside the Kubernetes cluster and is used to run kubectl/calicoctl commands. This node also runs a standalone instance of bird and is used for bgppeering in the relevant lab modules fucntioning as an upstream router. Finally, since this node is outside the cluster, it is used to simulate external connectivity when needed.

Followings are the CIDRs used to build the training lab environment.

* 10.0.1.0/24 host CIDR
* 10.48.0.0/16 Kubernetes Pod Network (via kubeadm --pod-network-cidr)
* 10.49.0.0/16 Kubernetes Service CIDR (via kubeadm --service-cidr)

## SSH access to the nodes

To ssh into the lab instances, type the following commands. The associated IP addresses are included in the lab setup diagram.

* ssh control1
* ssh worker1
* ssh worker2

## Lab modules

[1.Install Calico Enterprise](./1.Install-Calico-Enterprise/README.md)

[2.Implement pod networking using Calico Enterprise CNI and IPAM](./2.Implement-pod-networking-using-Calico-Enterprise-CNI-and-IPAM/README.md)

[3.Cross Node Connectivity](./3.Cross-Node-Connectivity/README.md)

[4.Kubernetes Services and CE Service Advertisement](./4.Kubernetes-Services-and-CE-Service-Advertisement/README.md)

[5.Calico Enterprise Egress Gateway](./5.Calico-Enterprise-Egress-Gateway/README.md)

[6.Calico Ingress Gateway](./6.Calico-Ingress-Gateway/README.md)

[7.Implement Calico Enterprise eBPF](./7.Implement-Calico-Enterprise-eBPF/README.md)
