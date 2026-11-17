# Kubernetes Networking Training

This is the hands-on lab guide for Calico Networking Training.

---

## Lab setup

The following diagram depicts the lab setup used for the entire training.

<a href="img/lab-topology.svg"><img src="img/lab-topology.svg" alt="Lab topology" width="1200"></a>

The following is a short description of each node in the lab.

* **Control1** is the Kubernetes control-plane node and runs the Kubernetes control plane pods.
* **Worker1** is a Kubernetes worker node.
* **Worker2** is a Kubernetes worker node.
* **Bastion** is a node outside the Kubernetes cluster and is used to run kubectl/calicoctl commands. This node also runs a standalone instance of bird and is used for BGP peering in the relevant lab modules, functioning as an upstream router. Finally, since this node is outside the cluster, it is used to simulate external connectivity when needed.

The following CIDRs are used to build the training lab environment.

| CIDR | Purpose |
|---|---|
| `10.0.1.0/24` | Host CIDR |
| `10.48.0.0/16` | Kubernetes Pod Network (via kubeadm `--pod-network-cidr`) |
| `10.49.0.0/16` | Kubernetes Service CIDR (via kubeadm `--service-cidr`) |

---

## SSH access to the nodes

To ssh into the lab instances, type the following commands. The associated IP addresses are included in the lab setup diagram.

```bash
ssh control1
ssh worker1
ssh worker2
```

The Calico Enterprise version used throughout the training is set once, as the `CE_VERSION` variable in step 1 of module 1. Every later command that needs the version reads that variable, so do not retype the version anywhere else.

---

## Lab modules

| # | Module | What you do |
|---|---|---|
| 1 | [Install Calico Enterprise](./1.Install-Calico-Enterprise/README.md) | Install Calico Enterprise and calicoctl, and deploy the yaobank sample application |
| 2 | [Implement pod networking using Calico Enterprise CNI and IPAM](./2.Implement-pod-networking-using-Calico-Enterprise-CNI-and-IPAM/README.md) | Examine pod network connectivity and create an additional IPPool |
| 3 | [Cross Node Connectivity](./3.Cross-Node-Connectivity/README.md) | Use VXLAN mode, configure an externally routable IPPool, and peer with an upstream BGP router |
| 4 | [Kubernetes Services and CE Service Advertisement](./4.Kubernetes-Services-and-CE-Service-Advertisement/README.md) | Explore service iptables rules and advertise service IPs over BGP |
| 5 | [Calico Enterprise Egress Gateway](./5.Calico-Enterprise-Egress-Gateway/README.md) | Give traffic leaving the cluster a fixed source identity |
| 6 | [Calico Ingress Gateway](./6.Calico-Ingress-Gateway/README.md) | Build a shared Gateway with TLS passthrough, TLS termination, HTTP redirect and canary patterns |
| 7 | [Implement Calico Enterprise eBPF](./7.Implement-Calico-Enterprise-eBPF/README.md) | Enable, inspect and troubleshoot the eBPF dataplane, then revert to iptables |
