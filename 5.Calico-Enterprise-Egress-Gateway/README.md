# In this lab

This lab provides the instructions to:

* [Overview](#overview)
* [Implement Calico Enterprise Egress Gateway](#implement-calico-enterprise-egress-gateway)

### Overview

In Kubernetes, Ingress traffic refers to any traffic that is initiated from outside the cluster to the services running inside the Kubernetes cluster. Egress traffic is just the opposite, any traffic that is initiated from the pods from within the cluster to the IP endpoints that are located outside the cluster. Kubernetes provides a native ingress resource to manage and control Ingress traffic. However, there is no native Kubernetes Egress resource. So when pods need to connect to an endpoint outside the cluster, they do so using their own IP addresses by default. Considering that an application could be implemented through one or more pods and the fact that pods in Kubernetes are ephemeral, it is almost impossible to identify and control the Kubernetes egress traffic from outside the cluster as the IP addresses are constantly changing. Egress gateways (EGWs) are pods that act as gateways for traffic leaving the cluster from certain client pods. The primary function of egress gateway is to configure the egress gateway client to have a particular and persistent source IP address when connecting to services outside the Kubernetes cluster.

After finshing this lab, you should gain a good understanding of how to deploy Calico Enterprise Egress Gateway and establish a permanent identity for the traffic that is leaving the cluster.

______________________________________________________________________________________________________________________________________________________________________

### Implement Calico Enterprise Egress Gateway

#### Enable egress gateway support in FelixConfiguration

Enable egress gateway support by patching FelixConfiguration to support egress gateway both per namespace and per pod.

```bash
kubectl patch felixconfiguration.p default --type='merge' -p '{"spec":{"egressIPSupport":"EnabledPerNamespaceOrPerPod"}}'
```

#### Create the egress gateway IPPool

Egress gateways use the IPPool resource for a particular application when it connects outside of the cluster. Run the following command to create the egress gateway IPPool.

* `blockSize: 32` allocates the pool one IP address at a time, so each egress gateway pod gets its own address.
* `nodeSelector: "!all()"` stops Calico from using this pool for ordinary pods; only pods that explicitly ask for it (the egress gateways) get addresses from it.

```yaml
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: egress-ippool-1
spec:
  cidr: 10.10.10.0/30
  blockSize: 32
  nodeSelector: "!all()"
EOF
```

#### Deploy the egress gateway client application

Let's create a namespace and application, which will be using egress gateway to connect to resources outside the cluster.

```yaml
kubectl apply -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: app1
  labels:
    tenant: tenant1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
  namespace: app1
  labels:
    app: app1
    tenant: tenant1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
      tenant: tenant1
  template:
    metadata:
      labels:
        app: app1
        tenant: tenant1
    spec:
      containers:
      - name: app1
        image: praqma/network-multitool
        env:
        - name: HTTP_PORT
          value: "1180"
        - name: HTTPS_PORT
          value: "11443"
        ports:
        - containerPort: 1180
          name: http-port
        - containerPort: 11443
          name: https-port
        resources:
          requests:
            cpu: "1m"
            memory: "20Mi"
          limits:
            cpu: "10m"
            memory: "20Mi"
EOF
```

#### Verify the client pods are running

Make sure the pods are running.

```bash
kubectl get pods -n app1
```

#### Deploy the egress gateway

Egress gateways are deployed with an `EgressGateway` resource. The Tigera operator turns it into a Deployment of egress gateway pods, and takes care of the details for you:

* it uses the egress gateway image that matches the Calico Enterprise version installed in the cluster;
* it copies the image pull secret into the egress gateway's namespace;
* it enables the Policy Sync API in the default FelixConfiguration, which egress gateways use for symmetric routing.

Note the following configurations:

* `ipPools` is the IPPool that provides IP addresses to the egress gateway pods: the egress gateway IPPool you created above.
* `egress-code: red` is a label on the egress gateway pods. The client namespace selects the egress gateway by this label in a later step.
* `replicas: 1` deploys a single egress gateway pod. The number of replicas must not exceed the number of free IP addresses in the IPPool.

For the full list of options, see [Deploy a group of egress gateways](https://docs.tigera.io/calico-enterprise/latest/networking/egress/egress-gateway-on-prem#deploy-a-group-of-egress-gateways) in the Calico Enterprise documentation.

```yaml
kubectl apply -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: EgressGateway
metadata:
  name: egress-gateway
  namespace: default
spec:
  logSeverity: "Info"
  replicas: 1
  ipPools:
  - cidr: "10.10.10.0/30"
  template:
    metadata:
      labels:
        egress-code: red
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      terminationGracePeriodSeconds: 0
EOF
```

#### Verify the egress gateway pod is running

Make sure that the egress gateway pod is running and has an IP address from the egress gateway IPPool (`10.10.10.0/30`). The operator names the Deployment after the `EgressGateway` resource.

```bash
kubectl get egressgateway,deployment -n default
kubectl get pods -n default -l egress-code=red -o wide
```

```bash
NAME                                              AGE
egressgateway.operator.tigera.io/egress-gateway   49s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/egress-gateway   1/1     1            1           49s
NAME                             READY   STATUS    RESTARTS   AGE   IP           NODE                                         NOMINATED NODE   READINESS GATES
egress-gateway-5dc699c54-zs92f   1/1     Running   0          49s   10.10.10.2   ip-10-0-1-31.ca-central-1.compute.internal   <none>           <none>
```

#### Annotate the client namespace to use the egress gateway

We will now need to configure our egress gateway client (app1 application) to use the egress gateway. We could configure specific pods/deployments in app1 namespace to use the egress gateway by annotating the pods/deployment or all the pods/deployments in app1 namespace by annotating the namespace. Since we only have a single app in app1 namespace, we will configure the annotation on the namespace.

```bash
kubectl annotate ns app1 egress.projectcalico.org/selector="egress-code == 'red'"
```

#### Add a namespace selector annotation

By default, the above selector can only match egress gateways in the same namespace. To select gateways in a different namespace, specify a namespaceSelector annotation as well. Since our egress gateway pod is running in `default` namespace and our egress gateway client is running in `app1` namespace, we need to run the following command.

```bash
kubectl annotate ns app1 egress.projectcalico.org/namespaceSelector="projectcalico.org/name == 'default'"
```

#### Verify the egress gateway annotations

Confirm that the `app1` namespace carries both annotations. Every pod in `app1` now sends its traffic leaving the cluster through the egress gateway pods labeled `egress-code: red` in the `default` namespace.

```bash
kubectl get ns app1 -o jsonpath='{.metadata.annotations.egress\.projectcalico\.org/selector}{"\n"}{.metadata.annotations.egress\.projectcalico\.org/namespaceSelector}{"\n"}'
```

```bash
egress-code == 'red'
projectcalico.org/name == 'default'
```

#### Deploy the BGPConfiguration and BGPPeer

Deploy the needed BGPConfiguration and BGPPeer so we route our traffic to the `bastion` host through the egress gateway.

```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-64512
spec:
  peerIP: 10.0.1.10
  asNumber: 64512
EOF
```

```yaml
kubectl patch bgpconfigurations default --type=merge -p '
spec:
  serviceClusterIPs:
  - cidr: 10.49.0.0/16
  communities:
  - name: bgp-large-community
    value: 64512:120
  prefixAdvertisements:
  - cidr: 10.10.10.0/31
    communities:
    - bgp-large-community
    - 64512:120
'
```

#### Verify BGP sessions and update the bird config on the bastion

`Bastion` host is simulating our upstream router and we should have BGP sessions established to all the cluster nodes using the above configurations. Run the following command on the `bastion` node to validate the bgp sessions from the `bastion` node to the cluster nodes.

```bash
sudo birdc show protocols
```

```bash
BIRD 1.6.8 ready.
name     proto    table    state  since       info
direct1  Direct   master   up     23:42:33    
kernel1  Kernel   master   up     23:42:33    
device1  Device   master   up     23:42:33    
control1 BGP      master   up     23:42:35    Established   
worker1  BGP      master   up     23:42:36    Established   
worker2  BGP      master   up     23:42:35    Established
```

We must also set the bird config on the bastion to accept advertisements of the egress gateway network. The bastion only installs routes that its import filter `rt_import` accepts, so we add the egress gateway IPPool `10.10.10.0/30` to that filter. The new line must come before `reject;`, because BIRD stops at the first `accept` or `reject` that matches.

This is the change we make in `/etc/bird/bird.conf` (the new line is marked with an arrow):

```bash
# Import filter
filter rt_import {
                        if (net ~ 10.48.2.0/24) then accept;
                        if (net ~ 10.49.0.0/16) then accept;
                        if (net ~ 10.50.0.0/24) then accept;
                        if (net ~ 10.10.10.0/30) then accept; <-------
                        reject;
        }
```

Run the following command to add the line right after the `10.50.0.0/24` line. The `grep` check skips the change if the line is already there, so the command is safe to run more than once.

```bash
sudo grep -q '10.10.10.0/30' /etc/bird/bird.conf || \
  sudo sed -i '/if (net ~ 10.50.0.0\/24) then accept;/a\                        if (net ~ 10.10.10.0/30) then accept;' /etc/bird/bird.conf
```

Verify the import filter now includes the egress gateway network:

```bash
sudo sed -n '/filter rt_import/,/}/p' /etc/bird/bird.conf
```

```bash
filter rt_import {
                        if (net ~ 10.48.2.0/24) then accept;
                        if (net ~ 10.49.0.0/16) then accept;
                        if (net ~ 10.50.0.0/24) then accept;
                        if (net ~ 10.10.10.0/30) then accept;
                        reject;
        }
```

> **Note:** Alternatively, you can make the same change by hand with `sudo vi /etc/bird/bird.conf`.

Check the configuration is valid, then restart bird. If the check fails, bird is not restarted and keeps running with its current configuration.

```bash
sudo bird -p -c /etc/bird/bird.conf && sudo systemctl restart bird
```

Confirm the BGP sessions to the cluster nodes are `Established` again:

```bash
sudo birdc show protocols
```

#### Check the routes on the bastion node

Check the routes on the `bastion` node. You should see that the edge gateway pod is reachable through the worker node where it has been deployed.

```bash
ip route
```

```bash
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.10.10.0 via 10.0.1.31 dev ens5 proto bird 
10.48.2.216/29 via 10.0.1.31 dev ens5 proto bird 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1
```

#### Start a netcat listener on the bastion

Let verify our egress gateway configurations in action and ensure that `app1` IP address are proxied by egress gateway when connecting to services outside the cluster. Open a second terminal to your lab instance. On that terminal, start netcat on the `bastion` host to listen to a specific port.

```bash
netcat -nvlkp 7777
```

#### Exec into an app1 pod

On the original terminal window, exec into any of the pods in the app1 namespace.

```bash
APP1_POD=$(kubectl get pod -n app1 --no-headers -o name | head -1) && echo $APP1_POD
```

```bash
kubectl exec -ti $APP1_POD -n app1 -- sh
```

#### Connect to the bastion port from the pod

Try to connect to the port in the bastion host.

```bash
nc -zv 10.0.1.10 7777
```

#### Exit the pod shell

Type `exit` to exit out the pod terminal.

```bash
exit
```

#### Confirm the connection came from the egress gateway IP

Go to the terminal that you ran the following netcat server on the `bastion` node. You should see an output saying you connected from the IP of the egress gateway pod to the netcat server. Note that the egress gateway IP address could be different in your environment.

```bash
tigera@bastion:~$ netcat -nvlkp 7777
Listening on 0.0.0.0 7777
Connection received on 10.10.10.0 39457tigera@bastion:~$ netcat -nvlkp 7777
Listening on 0.0.0.0 7777
Connection received on 10.10.10.0 39457
```

Stop the netcat listener process in the `bastion` host with `^C`.

#### Check where the egress gateway client pods are running

Let's check on the special routing that Felix has implemented to proxy the `app1` traffic through the egress gateway. Check where the egress gateway client pods are running.

```bash
kubectl get pods -n app1 -o wide
```

```bash
NAME                               READY   STATUS    RESTARTS   AGE     IP            NODE                                      NOMINATED NODE   READINESS GATES
app1-deployment-6ccc9654cb-c8ztq   1/1     Running   0          3h50m   10.48.0.60    ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
app1-deployment-6ccc9654cb-wjnfl   1/1     Running   0          3h50m   10.48.0.210   ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
```

#### Log into the worker node running the client pod

Log into the worker node where the egress gateway client pod `app1` is running. In this case, there are two pods and each one is running on a different worker node. Choose a node where the `app1` is running on and ssh into that node. Note `app1` pod IP address running on worker2. It is `10.48.0.60` in this case.

```bash
ssh worker2
```

#### Observe the routing policy for the client pod

Observe the routing policy that is programmed for the the `app1` workload POD IP address `10.48.0.60`.

```bash
ip rule
```

```bash
0:      from all lookup local
100:    from 10.48.0.60 fwmark 0x80000/0x80000 lookup 250
32766:  from all lookup main
32767:  from all lookup default
```

#### Confirm the egress gateway is the next hop

Confirm that the policy is choosing the egress gateway as the next hop for any source traffic from `app1`.

> **Note:** ensure to use the correct table number with the following command. In this case, the table number is 250.

```bash
ip route show table 250
```

```bash
default via 10.10.10.0 dev egress.calico onlink 
```

#### Configure an ICMP readiness probe on the egress gateway

Egress gateways support HTTP and ICMP probes for readiness health checking. If all probes of one type (ICMP or HTTP) fail, the egress gateway pod failing the probe goes non-Ready. Let's configure our egress gateway to ping the bastion node `10.0.1.10` for its readiness check. Should this ping fail, the egress gateway pod should go non-ready.

Probes are configured on the `EgressGateway` resource under `egressGatewayFailureDetection`, not on the Deployment: the operator owns the Deployment and overwrites manual edits to it. Patch the `EgressGateway` resource to add an ICMP probe to `10.0.1.10`. The operator rolls out the change, so the egress gateway pod restarts.

```bash
kubectl patch egressgateway egress-gateway -n default --type=merge -p '{"spec":{"egressGatewayFailureDetection":{"icmpProbe":{"ips":["10.0.1.10"]}}}}'
```

Verify the probe settings. `intervalSeconds` and `timeoutSeconds` were not set, so they take their default values of 5 and 15 seconds.

```bash
kubectl get egressgateway egress-gateway -n default -o jsonpath='{.spec.egressGatewayFailureDetection.icmpProbe}{"\n"}'
```

```bash
{"intervalSeconds":5,"ips":["10.0.1.10"],"timeoutSeconds":15}
```

#### Confirm the egress gateway pod is running

Make sure that the new egress gateway pod is running and ready by running the following command.

```bash
kubectl get pods -n default -l egress-code=red -o wide
```

#### Block ICMP traffic to the bastion with a network policy

Let's now implement a security policy that blocks ICMP traffic from egress gateway pod to `10.0.1.10`, but allows every other traffic.

```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: default.egress-gateway-allow-everything-deny-ping-to-bastion
  namespace: default
spec:
  tier: default
  selector: egress-code == "red"
  serviceAccountSelector: ''
  egress:
    - action: Deny
      protocol: ICMP
      source: {}
      destination:
        nets:
          - 10.0.1.10/32
    - action: Allow
      source: {}
      destination: {}
  types:
    - Egress
EOF
```

#### Observe the egress gateway becomes non-ready

The ICMP probe `timeoutSeconds` is `15` by default. It should take about 15-20s after implementing the above policy for the egress gateway pod to show as non-ready as depicted below.

```bash
watch kubectl get pods -n default -l egress-code=red -o wide
```

```bash
NAME                              READY   STATUS    RESTARTS   AGE     IP           NODE                                         NOMINATED NODE   READINESS GATES
egress-gateway-5fb89cf946-snznd   0/1     Running   0          3m25s   10.10.10.0   ip-10-0-1-31.ca-central-1.compute.internal   <none>           <none>
```

Press `Ctrl+C` to stop `watch`.

> **Note:** Felix on each node also checks the readiness of every egress gateway and stops sending traffic through one that is not ready. With no ready egress gateway left, traffic from `app1` to destinations outside the cluster is dropped rather than sent from the pod's own IP address, so the source IP that outside systems see never changes. It flows again through the egress gateway as soon as the gateway is ready again.

#### Clean up the lab resources

Clean up the resources that were deployed for the purpose of this lab.

```bash
kubectl delete ns app1
```

```bash
kubectl delete networkpolicies.p default.egress-gateway-allow-everything-deny-ping-to-bastion -n default
```

```bash
kubectl delete egressgateway egress-gateway -n default
```

> **Congratulations! You have completed `5. Calico Enterprise Egress Gateway` lab.**
