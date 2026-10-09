# Module 4: Kubernetes Services and CE Service Advertisement

## In this lab

This lab provides the instructions to:

* [Overview](#overview)
* [Explore Kubernetes service ClusterIP iptables rules](#explore-kubernetes-service-clusterip-iptables-rules)
* [Explore Kubernetes service NodePort iptables rules](#explore-kubernetes-service-nodeport-iptables-rules)
* [Advertise the ServiceIP range using Calico Enterprise](#advertise-the-serviceip-range-using-calico-enterprise)
* [Advertise services with `externalTrafficPolicy: Local`to preserve the source IP and avoid extra hop in routing](#advertise-services-with-externaltrafficpolicy-local-to-preserve-the-source-ip-and-avoid-extra-hop-in-routing)
* [Clean up](#clean-up)

---

## Overview

Kubernetes is a dynamic environment and pods are not permanent resources. Pods are created and destroyed to match the desired state of your cluster. Since Pods and their IP addresses are ephemeral resources, Kubernetes provides a virtual entity called service as a permanent point of network communication between Pods. Once a service is created, it will stay there until deleted. Services provide a dns name and virtual IP address. In the context of Kubernetes service, pods are called endpoints. Kubernetes uses service controller, which it turn uses labels and selectors to keep the service endpoints updated as pods are created and destroyed.

By default, Kubernetes service cluster IPs are accessible only within the cluster and external access to the service requires a dedicated load balancer or ingress controller. Calico Enterprise enables you to advertise Kubernetes service IPs outside the cluster. Calico Enterprise supports advertising a service’s cluster IPs and external IPs. In cases where a service’s cluster IP is not routable, the service can be accessed using its external IP.

After finishing this lab, you should gain a good understanding of Kubernetes services and how to use Calico Enterprise to advertise Kubernetes services.

---

## Explore Kubernetes service ClusterIP iptables rules

There are four types of Kubernetes services. ClusterIP, NodePort, LoadBalancer, and ExternalName. kube-proxy is the Kubernetes service agent that is responsible for implementing Kubernetes services definitions in a Kubernetes environment. In this section, we will explore how kube-proxy uses iptables to implement Kubernetes service `ClusterIP`. `ClusterIP` service is the default service type and is used to expose a service on an IP address that is internal to the cluster.

### 1. SSH into worker1

We will run this lab from `worker1` so we can explore the iptables rules that kube-proxy has set up. Run the following command to ssh into `worker1`.

```bash
ssh worker1
```

### 2. Review the yaobank services and pods

Let's take a look at the services and pods in the `yaobank` kubernetes namespace. We should have three services deployed. One `NodePort` service and two `ClusterIP` services.

```bash
kubectl get svc -n yaobank
```

**Example output:**

```text
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
customer   NodePort    10.49.60.52    <none>        80:30180/TCP   2d2h
database   ClusterIP   10.49.164.49   <none>        2379/TCP       2d2h
summary    ClusterIP   10.49.80.32    <none>        80/TCP         2d2h
```

### 3. Find the service endpoints

Run the following command to find the endpoints for each one of the services in yaobank namespace.

```bash
kubectl get endpoints -n yaobank
```

**Example output:**

```text
NAME       ENDPOINTS                         AGE
customer   10.48.0.43:80                     2d2h
database   10.48.128.0:2379                  2d2h
summary    10.48.128.1:80,10.48.128.192:80   2d2h
```

### 4. List the yaobank pods

List the pods in the yaobank namespace.

```bash
kubectl get pods -n yaobank -o wide
```

**Example output:**

```text
NAME                        READY   STATUS    RESTARTS   AGE    IP              NODE                                      NOMINATED NODE   READINESS GATES
customer-687b8d8f74-tcclp   1/1     Running   0          2d2h   10.48.0.43      ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
database-75ccfdc84f-hqr64   1/1     Running   0          2d2h   10.48.128.0     ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
summary-7d78c9976b-sxv4k    1/1     Running   0          2d2h   10.48.128.192   ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
summary-7d78c9976b-wjdd5    1/1     Running   0          2d2h   10.48.128.1     ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
```

You can see that the IP addresses listed as the service endpoints in the previous step map to the backing pods in the yaobank namespace as expected. Each service is backed by one or more pods spread across the nodes in the cluster.

### 5. Find the summary service endpoints

Let's explore the iptables rules that implement the `summary` service. Let's start by finding the service endpoints for the `summary` `ClusterIP` service.

```bash
kubectl get endpoints -n yaobank summary
```

**Example output:**

```text
NAME      ENDPOINTS                         AGE
summary   10.48.128.1:80,10.48.128.192:80   2d2h
```

### 6. Examine the KUBE-SERVICES chain

As per the previous output, the `summary` service has two endpoints (`10.48.128.1` on port `80` AND `10.48.128.192` on port `80` in this example output). Starting from the `KUBE-SERVICES` iptables chain, we will traverse each chain until you get to the rule directing traffic to these endpoint IP addresses.

Let's examine the KUBE-SERVICE chain.

```bash
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | column -t
```

**Example output:**

```text
Chain  KUBE-SERVICES  (2                         references)
pkts   bytes          target                     prot         opt  in  out  source     destination
8      480            KUBE-SVC-3NJEN5CBT7DGINLD  tcp          --   *   *    0.0.0.0/0  10.49.69.235   /*  tigera-elasticsearch/tigera-secure-es-gateway-http:es-gateway-elasticsearch-port  cluster  IP          */     tcp   dpt:9200
104    12362          KUBE-SVC-TCOU7JCQXEZGVUNU  udp          --   *   *    0.0.0.0/0  10.49.0.10     /*  kube-system/kube-dns:dns                                                          cluster  IP          */     udp   dpt:53
0      0              KUBE-SVC-P37GYSV6HJH3GAT2  tcp          --   *   *    0.0.0.0/0  10.49.241.76   /*  tigera-packetcapture/tigera-packetcapture:tigera-packetcapture                    cluster  IP          */     tcp   dpt:443
0      0              KUBE-SVC-P7GI5BPQ3GTNT5FK  tcp          --   *   *    0.0.0.0/0  10.49.105.153  /*  tigera-prometheus/calico-node-alertmanager:web                                    cluster  IP          */     tcp   dpt:9093
0      0              KUBE-SVC-3XH7Z4WONXPKZ54M  tcp          --   *   *    0.0.0.0/0  10.49.146.237  /*  tigera-kibana/tigera-secure-kb-http:https                                         cluster  IP          */     tcp   dpt:5601
0      0              KUBE-SVC-PX5FENG4GZJTCELT  tcp          --   *   *    0.0.0.0/0  10.49.60.52    /*  yaobank/customer:http                                                             cluster  IP          */     tcp   dpt:80
0      0              KUBE-SVC-JD5MR3NA4I4DYORP  tcp          --   *   *    0.0.0.0/0  10.49.0.10     /*  kube-system/kube-dns:metrics                                                      cluster  IP          */     tcp   dpt:9153
0      0              KUBE-SVC-BPJNZGPODTH4UZQI  tcp          --   *   *    0.0.0.0/0  10.49.199.67   /*  calico-system/calico-node-metrics:calico-metrics-port                             cluster  IP          */     tcp   dpt:9081
0      0              KUBE-SVC-KQVGIOWQAVNMB2ZL  tcp          --   *   *    0.0.0.0/0  10.49.22.249   /*  calico-system/calico-kube-controllers-metrics:metrics-port                        cluster  IP          */     tcp   dpt:9094
0      0              KUBE-SVC-FQBB24OXOBQUPC7L  tcp          --   *   *    0.0.0.0/0  10.49.75.250   /*  tigera-prometheus/prometheus-http-api:web                                         cluster  IP          */     tcp   dpt:9090
7      420            KUBE-SVC-27GAF2D4QQPTKE7C  tcp          --   *   *    0.0.0.0/0  10.49.31.204   /*  tigera-elasticsearch/tigera-secure-es-http:https                                  cluster  IP          */     tcp   dpt:9200
0      0              KUBE-SVC-HWMH37Q2Y4HJ5UPP  tcp          --   *   *    0.0.0.0/0  10.49.160.225  /*  tigera-fluentd/fluentd-metrics:fluentd-metrics-port                               cluster  IP          */     tcp   dpt:9081
0      0              KUBE-SVC-AE2X4VPDA5SRYCA6  tcp          --   *   *    0.0.0.0/0  10.49.164.49   /*  yaobank/database:http                                                             cluster  IP          */     tcp   dpt:2379
0      0              KUBE-SVC-BXX6NV5PBDEKW23Y  tcp          --   *   *    0.0.0.0/0  10.49.114.47   /*  tigera-system/tigera-api:queryserver                                              cluster  IP          */     tcp   dpt:8080
0      0              KUBE-SVC-OIQIZJVJK6E34BR4  tcp          --   *   *    0.0.0.0/0  10.49.80.32    /*  yaobank/summary:http                                                              cluster  IP          */     tcp   dpt:80
63     3780           KUBE-SVC-ERIFXISQEP7F7OF4  tcp          --   *   *    0.0.0.0/0  10.49.0.10     /*  kube-system/kube-dns:dns-tcp                                                      cluster  IP          */     tcp   dpt:53
0      0              KUBE-SVC-VWTJDZUOIAKKPMUV  tcp          --   *   *    0.0.0.0/0  10.49.251.45   /*  tigera-compliance/compliance:compliance-api                                       cluster  IP          */     tcp   dpt:443
0      0              KUBE-SVC-YFF642K22PZPYCSR  tcp          --   *   *    0.0.0.0/0  10.49.191.172  /*  tigera-elasticsearch/tigera-elasticsearch-metrics:metrics-port                    cluster  IP          */     tcp   dpt:9081
0      0              KUBE-SVC-5YT3S4Q5ZQB7MXPI  tcp          --   *   *    0.0.0.0/0  10.49.114.47   /*  tigera-system/tigera-api:apiserver                                                cluster  IP          */     tcp   dpt:443
0      0              KUBE-SVC-RK657RLKDNVNU64O  tcp          --   *   *    0.0.0.0/0  10.49.179.78   /*  calico-system/calico-typha:calico-typha                                           cluster  IP          */     tcp   dpt:5473
0      0              KUBE-SVC-ZMPNACNGKBKCFXCW  tcp          --   *   *    0.0.0.0/0  10.49.199.67   /*  calico-system/calico-node-metrics:calico-bgp-metrics-port                         cluster  IP          */     tcp   dpt:9900
0      0              KUBE-SVC-LHFU7GY5JD5I4Q3O  tcp          --   *   *    0.0.0.0/0  10.49.47.211   /*  tigera-elasticsearch/tigera-secure-es-internal-http:https                         cluster  IP          */     tcp   dpt:9200
0      0              KUBE-SVC-TQCARI64B4Y6IXTV  tcp          --   *   *    0.0.0.0/0  10.49.69.235   /*  tigera-elasticsearch/tigera-secure-es-gateway-http:es-gateway-kibana-port         cluster  IP          */     tcp   dpt:5601
0      0              KUBE-SVC-HNOWWVZB7NMFCWE4  tcp          --   *   *    0.0.0.0/0  10.49.246.160  /*  tigera-intrusion-detection/anomaly-detection-api:anomaly-detection-api-https      cluster  IP          */     tcp   dpt:8080
0      0              KUBE-SVC-QJP4DZODROFX2FWR  tcp          --   *   *    0.0.0.0/0  10.49.194.132  /*  tigera-manager/tigera-manager                                                     cluster  IP          */     tcp   dpt:9443
0      0              KUBE-SVC-NPX46M4PTMTKRN6Y  tcp          --   *   *    0.0.0.0/0  10.49.0.1      /*  default/kubernetes:https                                                          cluster  IP          */     tcp   dpt:443
0      0              KUBE-SVC-EZYNCFY2F7N6OQA2  tcp          --   *   *    0.0.0.0/0  10.49.242.76   /*  ingress-nginx/ingress-nginx-controller-admission:https-webhook                    cluster  IP          */     tcp   dpt:443
321    19296          KUBE-NODEPORTS             all          --   *   *    0.0.0.0/0  0.0.0.0/0      /*  kubernetes                                                                        service  nodeports;  NOTE:  this  must      be  the  last  rule  in  this  chain  */  ADDRTYPE  match  dst-type  LOCAL
```

Each iptables chain consists of a list of rules that are executed in order until a rule matches. The key columns/elements to note in this output are:
* `target` - which chain iptables will jump to if the rule matches
* `prot` - the protocol match criteria
* `source`, and `destination` - the source and destination IP address match criteria
* the comments that kube-proxy includes
* the additional match criteria at the end of each rule - e.g `dpt:80` that specifies the destination port match

### 7. Inspect the summary service rule

Now let's look more closely at the rules for the `summary` service.

```bash
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | column -t | grep -E summary 
```

You should see an output similar to the following.

```text
0  0  KUBE-SVC-OIQIZJVJK6E34BR4  tcp  --  *  *  0.0.0.0/0  10.49.80.32  /*  yaobank/summary:http  cluster  IP  */  tcp  dpt:80
```

The rule above directs traffic destined for the summary service clusterIP (`10.49.80.32` in the example output) to the chain that load balances the service (KUBE-SVC-XXXXXXXXXXXXXXXX).

### 8. Examine the service load-balancing chain

`kube-proxy` in `iptables` mode uses an equal probability algorithm to load balance traffic between pods.  We currently have two `summary` pods.

Let's examine how this loadbalancing works among the Kubernetes service endpoints. (Remember your chain name may be different than this example.)

```bash
sudo iptables -v --numeric --table nat --list KUBE-SVC-OIQIZJVJK6E34BR4 | column -t
```

**Example output:**

```text
Chain  KUBE-SVC-OIQIZJVJK6E34BR4  (1                         references)
pkts   bytes                      target                     prot         opt  in  out  source         destination
0      0                          KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.80.32  /*  yaobank/summary:http  cluster  IP         */    tcp     dpt:80
0      0                          KUBE-SEP-NCOI6JAXVX2D4BXJ  all          --   *   *    0.0.0.0/0      0.0.0.0/0    /*  yaobank/summary:http  */       statistic  mode  random  probability  0.50000000000
0      0                          KUBE-SEP-TVZHKMXMBG2BIHXS  all          --   *   *    0.0.0.0/0      0.0.0.0/0    /*  yaobank/summary:http  */
```

Notice that `kube-proxy` uses `iptables` `statistic` module to set the probability for a packet to be randomly matched.  Make sure you scroll all the way to the right to see the full output.

The second rule directs traffic destined for the `summary` service to the chain that delivers packets to the first service endpoint (KUBE-SEP-NCOI6JAXVX2D4BXJ) with a probability of 0.50000000000.
The third rule unconditionally directs to the second service endpoint chain (KUBE-SEP-TVZHKMXMBG2BIHXS). The result is that traffic is load balanced across the service endpoints equally (on average).

If there were 3 service endpoints, then the first chain match would be probability of 0.33333333, the second probability of 0.5, and the last unconditional. The result is each service endpoint receives a third of the traffic (on average).

### 9. Inspect a service endpoint chain

Let's look at one of the service endpoint chains. (Remember your chain names may be different than this example.)

```bash
sudo iptables -v --numeric --table nat --list KUBE-SEP-NCOI6JAXVX2D4BXJ | column -t
```

**Example output:**

```text
Chain  KUBE-SEP-NCOI6JAXVX2D4BXJ  (1              references)
pkts   bytes                      target          prot         opt  in  out  source         destination
0      0                          KUBE-MARK-MASQ  all          --   *   *    10.48.128.192  0.0.0.0/0    /*  yaobank/summary:http  */
0      0                          DNAT            tcp          --   *   *    0.0.0.0/0      0.0.0.0/0    /*  yaobank/summary:http  */  tcp  to:10.48.128.192:80
```

The second rule performs the DNAT that changes the destination IP from the service's clusterIP to the IP address of the service endpoint backing pod (`10.48.128.192` in this example). After this, standard Linux routing can handle forwarding the packet like it would for any other packet.

**Recap**

You've just traced the kube-proxy iptables rules used to load balance traffic to `summary` pods exposed as a service of type `ClusterIP`.

In summary, for a packet being sent to a clusterIP:
* The KUBE-SERVICES chain matches on the clusterIP and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
* The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
* The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).

---

## Explore Kubernetes service NodePort iptables rules

`NodePort` service enables us to expose a service outside the cluster using a static port on the cluster nodes. To reach a NodePort service from outside the cluster, we just need to use any of the cluster nodes IP addresses along with a port on the node. The default port range for NodePort is 30000-32767, but this range can be changed or extended through the kube-apiserver configuration.

### 1. Find the customer service endpoints

Let's explore the iptables rules that implement the `customer` service. Find the service endpoints for `customer` `NodePort` service. Keep using `worker1` to follow the instructions.

```bash
kubectl get endpoints -n yaobank customer
```

**Example output:**

```text
NAME       ENDPOINTS       AGE
customer   10.48.0.43:80   2d2h
```

### 2. Locate the KUBE-NODEPORTS chain

The `customer` service has one endpoint (`10.48.0.43` on port `80` in this example output). Starting from the `KUBE-SERVICES` iptables chain, we will traverse each chain until you get to the rule directing traffic to this endpoint IP address.

> [!NOTE]
> The `KUBE-SERVICE` chain handles the matching for service types `ClusterIP` and `LoadBalancer`. At the end of `KUBE-SERVICE` chain, another custom chain `KUBE-NODEPORTS` will handle traffic for service type `NodePort`.

```bash
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep KUBE-NODEPORTS | column -t
```

**Example output:**

```text
80  4836  KUBE-NODEPORTS  all  --  *  *  0.0.0.0/0  0.0.0.0/0  /*  kubernetes  service  nodeports;  NOTE:  this  must  be  the  last  rule  in  this  chain  */  ADDRTYPE  match  dst-type  LOCAL
```

`match dst-type LOCAL` matches any packet with a local host IP as the destination. i.e. any address that is assigned to one of the host's interfaces.

### 3. Examine the KUBE-NODEPORTS chain rules

Now let's look more closely at the rules for the `customer` service.

```bash
sudo iptables -v --numeric --table nat --list KUBE-NODEPORTS | column -t
```

**Example output:**

```text
Chain  KUBE-NODEPORTS  (1                         references)
pkts   bytes           target                     prot         opt  in  out  source     destination
0      0               KUBE-SVC-PX5FENG4GZJTCELT  tcp          --   *   *    0.0.0.0/0  0.0.0.0/0    /*  yaobank/customer:http  */  tcp  dpt:30180
```

The rule above directs traffic destined for the `customer` service to the chain that load balances the service (KUBE-SVC-PX5FENG4GZJTCELT). `tcp dpt:30180` matches any packet with the destination port of tcp 30180 (the node port of the `customer` service).

### 4. Examine the service load-balancing chain

`kube-proxy` in `iptables` mode uses an equal probability algorithm to load balance traffic between pods.  We currently have one `customer` pod.

Let's examine how this loadbalancing works among the Kubernetes service endpoints. (Remember your chain name may be different than this example.)

```bash
sudo iptables -v --numeric --table nat --list KUBE-SVC-PX5FENG4GZJTCELT | column -t
```

**Example output:**

```text
Chain  KUBE-SVC-PX5FENG4GZJTCELT  (2                         references)
pkts   bytes                      target                     prot         opt  in  out  source         destination
0      0                          KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.60.52  /*  yaobank/customer:http  cluster  IP   */         tcp  dpt:80
0      0                          KUBE-MARK-MASQ             tcp          --   *   *    0.0.0.0/0      0.0.0.0/0    /*  yaobank/customer:http  */       tcp  dpt:30180
0      0                          KUBE-SEP-GNFCEQXJBZQ2TQAR  all          --   *   *    0.0.0.0/0      0.0.0.0/0    /*  yaobank/customer:http  */
```

As we only have a single backing pod for the `customer` service, there is no loadbalancing to do, so there is a single rule (last rule) that directs all traffic to the chain that delivers the packet to the service endpoint (KUBE-SEP-XXXXXXXXXXXXXXXX).

### 5. Inspect the customer service endpoint chain

Finally, let's look into the KUBE-SEP-XXXXXXXXXXXXXXXX to `customer` endpoint. This rule delivers the packet to the `customer` service endpoint.

```bash
sudo iptables -v --numeric --table nat --list KUBE-SEP-GNFCEQXJBZQ2TQAR | column -t
```

**Example output:**

```text
Chain  KUBE-SEP-GNFCEQXJBZQ2TQAR  (1              references)
pkts   bytes                      target          prot         opt  in  out  source      destination
0      0                          KUBE-MARK-MASQ  all          --   *   *    10.48.0.43  0.0.0.0/0    /*  yaobank/customer:http  */
0      0                          DNAT            tcp          --   *   *    0.0.0.0/0   0.0.0.0/0    /*  yaobank/customer:http  */  tcp  to:10.48.0.43:80
```

The second rule performs the DNAT that changes the destination IP from the service's NodePort to the IP address of the service endpoint backing pod (`10.48.0.43` in this example). After this, standard Linux routing can handle forwarding the packet like it would for any other packet.

### 6. Exit the worker node

```bash
exit
```

**Recap**

You've just traced the kube-proxy iptables rules used to load balance traffic to `customer` pods exposed as a service of type `NodePort`.

In summary, for a packet being sent to a NodePort:

* The end of the KUBE-SERVICES chain jumps to the KUBE-NODEPORTS chain
* The KUBE-NODEPORTS chain matches on the NodePort and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
* The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
* The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).

---

## Advertise the ServiceIP range using Calico Enterprise

Calico Enterprise BGP service advertisement allows you to directly access the service without using a NodePorts or LoadBalancer service or a cluster Ingress Controller.

### 1. Check the routes on the bastion node

let's start by taking a look at the state of routes on the `bastion` node. If you completed the previous lab correctly, you should see one route that was learned from Calico that provides access to the nginx pod that was created in the externally routable namespace (the route ending in `proto bird` in this example output). In this section, we will advertise Kubernetes services rather than individual pods over BGP.

```bash
ip route
```

**Example output:**

```text
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.0.2 via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 metric 100 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.88/29 via 10.0.1.31 dev ens5 proto bird 
```

### 2. Confirm there are no BGP configurations

Run the following command and ensure that there is no bgpconfigurations in your cluster.

> [!NOTE]
> A BGPConfiguration resource (BGPConfiguration) represents BGP specific configuration options for the cluster or a specific node. The resource with the name `default` has a specific meaning and contains the BGP global default configuration. By default, there is no BGPConfiguration resource deployed in the cluster. However, Once BGP is activated, Calico has default built-in configurations for BGP in its data model. For example, it use 64512 as the default AS number.

```bash
kubectl get bgpconfigurations
```

You should receive the following output.

```text
NAME      CREATED AT
default   2026-10-05T18:27:40Z
```

### 3. Apply the default BGP configuration

Examine the following default BGP configuration and then apply it.

```bash
kubectl patch bgpconfigurations default --type=merge -p '{"spec":{"serviceClusterIPs":[{"cidr":"10.49.0.0/16"}]}}'
```

The `serviceClusterIPs` parameter tells Calico to advertise the cluster IP range.

### 4. Verify the BGPConfiguration

Verify the BGPConfiguration just implemented.

```bash
kubectl get bgpconfigurations default -o yaml
```

**Example output:**

```yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  creationTimestamp: "2026-10-05T18:27:40Z"
  generation: 2
  name: default
  resourceVersion: "620115"
  uid: 42d684af-2108-4e4b-ad35-59180842ca34
spec:
  logSeverityScreen: Info
  serviceClusterIPs:
  - cidr: 10.49.0.0/16
  serviceLoadBalancerAggregation: Enabled
```

### 5. Re-examine the routes on the bastion node

Now that we have enabled Service ClusterIP advertisement, let's examine the routes on `bastion` node again.

```bash
ip route
```

**Example output:**

```text
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.216/29 via 10.0.1.31 dev ens5 proto bird 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1
```

You should now see the cluster service cidr `10.49.0.0/16` advertised from each of the kubernetes cluster nodes. This means that traffic to any service's cluster IP address will get load-balanced across all nodes in the cluster by the network using ECMP (Equal Cost Multi Path). Kube-proxy then load balances the cluster IP across the service endpoints (backing pods) in exactly the same way as if a pod had accessed a service via a clusterIP from within the cluster.

### 6. Find the customer service cluster IP

Let's verify connectivity through the clusterIP. Find the cluster IP for the `customer` service.

```bash
kubectl get svc -n yaobank customer
```

**Example output:**

```text
NAME       TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
customer   NodePort   10.49.60.52   <none>        80:30180/TCP   2d3h
```

In this case, the ClusterIP is `10.49.60.52`. Your IP may be different.

### 7. Access the service through its cluster IP

Confirm you can access it from the `bastion` host.

```bash
curl 10.49.60.52
```

**Example output:**

```text
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <title>YAO Bank</title>
    <style>
    h2 {
      font-family: Arial, Helvetica, sans-serif;
    }
    h1 {
      font-family: Arial, Helvetica, sans-serif;
    }
    p {
      font-family: Arial, Helvetica, sans-serif;
    }
    </style>
  </head>
  <body>
        <h1>Welcome to YAO Bank</h1>
        <h2>Name: Spike Curtis</h2>
        <h2>Balance: 2389.45</h2>
        <p><a href="/logout">Log Out >></a></p>
  </body>
</html>
```

---

## Advertise services with `externalTrafficPolicy: Local` to preserve the source IP and avoid extra hop in routing

You can set `externalTrafficPolicy: Local` on a Kubernetes service to request that external traffic to a service only be routed via nodes, which have a local service endpoint (backing pod). This preserves the client source IP and avoids the second hop when kube-proxy loadbalances to a service endpoint (backing pod) on another node.

Traffic to the cluster IP for a service with `externalTrafficPolicy: Local` will be load-balanced across the nodes with endpoints for that service.
Note that `externalTrafficPolicy: Local` is only supported with service types of LoadBalancer and NodePort. For more information, visit the following link.

https://docs.tigera.io/networking/advertise-service-ips

### 1. Set externalTrafficPolicy to Local on the customer service

Let's start by updating the `customer` service to add `externalTrafficPolicy: Local`.

```bash
kubectl patch svc -n yaobank customer -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

**Example output:**

```text
service/customer patched
```

### 2. Examine the routes on the bastion node

Examine the routes on bastion node.

```bash
ip route
```

**Example output:**

```text
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.216/29 via 10.0.1.31 dev ens5 proto bird 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1 
10.49.60.52 via 10.0.1.31 dev ens5 proto bird 
```

### 3. Confirm the /32 service route

You should now have a `/32` route for the yaobank customer service (`10.49.60.52` in the above example output) advertised from the node hosting the customer service pod (`10.0.1.31` in this example output).

```bash
kubectl get pods -n yaobank -l app=customer -o wide
```

**Example output:**

```text
NAME                        READY   STATUS    RESTARTS   AGE    IP           NODE                                      NOMINATED NODE   READINESS GATES
customer-687b8d8f74-tcclp   1/1     Running   0          2d3h   10.48.0.43   ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
```

For each active service with `externalTrafficPolicy: Local`, Calico advertise the IP for that service as a `/32` route from the nodes that have endpoints for that service. This means that external traffic to the service will get load-balanced across all nodes in the cluster that have a service endpoint (backing pod) for the service by the network using ECMP (Equal Cost Multi Path). Kube-proxy then DNATs the traffic to the local backing pod on that node or load-balances equally to the local backing pods if there is more than one on the node.

The two main advantages of using `externalTrafficPolicy: Local` in this way are:
* There is a network efficiency win avoiding potential second hop of kube-proxy load-balancing to another node
* The client source IP addresses are preserved, which can be useful if you want to restrict access to a service to specific IP addresses using network policy applied to the backing pods

### 4. Verify the NodePort fails on a node without a local endpoint

In the previous labs, we accessed the yaobank frontend UI using curl from the `control1` node and we also mentioned that we can try any other cluster node IP address to hit the NodePort. As we've now set `externalTrafficPolicy: Local`, this will no longer work since there are no `customer` pods hosted on `control1`. Accessing the NodePort can only happen via the cluster node that is hosting the endpoint at this point.

The following should fail because of `externalTrafficPolicy: Local` configured on the `customer` service.

```bash
curl 10.0.1.20:30180
```

### 5. Verify the NodePort succeeds on the node hosting the endpoint

The following should go through. Please make sure to use the node IP address where customer pod is running.

```bash
curl 10.0.1.31:30180
```

**Example output:**

```text
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <title>YAO Bank</title>
    <style>
    h2 {
      font-family: Arial, Helvetica, sans-serif;
    }
    h1 {
      font-family: Arial, Helvetica, sans-serif;
    }
    p {
      font-family: Arial, Helvetica, sans-serif;
    }
    </style>
  </head>
  <body>
        <h1>Welcome to YAO Bank</h1>
        <h2>Name: Spike Curtis</h2>
        <h2>Balance: 2389.45</h2>
        <p><a href="/logout">Log Out >></a></p>
  </body>
```

---

## Clean up

Undo the changes made in this lab before moving on to the next lab.

### 1. Stop advertising the service cluster IP range

Undo the change you made to the `default` BGPConfiguration earlier in this lab. Setting `serviceClusterIPs` to `null` removes the field, so Calico stops advertising the service cluster IP range. The rest of the BGPConfiguration stays as it was.

```bash
kubectl patch bgpconfigurations default --type=merge -p '{"spec":{"serviceClusterIPs":null}}'
```

Verify the field is gone. The command should print an empty line:

```bash
kubectl get bgpconfigurations default -o jsonpath='{.spec.serviceClusterIPs}{"\n"}'
```

### 2. Delete the BGP peer

Delete the BGPPeer to the bastion node that you created in the previous lab.

```bash
kubectl delete bgppeers bgppeer-global-64512
```

### 3. Delete the yaobank namespace

The yaobank application is deployed again in a later lab. Delete its namespace for now.

```bash
kubectl delete namespace yaobank
```

> **Congratulations! You have completed `4. Kubernetes Services and CE Service Advertisement` lab.**

---

[← Previous: 3. Cross Node Connectivity](../3.Cross-Node-Connectivity/README.md) | [Training overview](../README.md) | [Next: 5. Calico Enterprise Egress Gateway →](../5.Calico-Enterprise-Egress-Gateway/README.md)
