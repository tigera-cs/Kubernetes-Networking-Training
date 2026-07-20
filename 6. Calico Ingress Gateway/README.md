# In this lab

This lab provides the instructions to:

* [Overview](#overview)
* [Enable Calico Ingress Gateway in host-network mode](#enable-calico-ingress-gateway-in-host-network-mode)
* [Deploy the shared Gateway behind the AWS load balancer](#deploy-the-shared-gateway-behind-the-aws-load-balancer)
* [Route HTTP traffic by host, path, and header](#route-http-traffic-by-host-path-and-header)
* [Split traffic for canary](#split-traffic-for-canary)
* [Terminate TLS at the Gateway](#terminate-tls-at-the-gateway)
* [Manipulate traffic with HTTPRoute filters](#manipulate-traffic-with-httproute-filters)
* [Self-service ingress with allowedRoutes and RBAC](#self-service-ingress-with-allowedroutes-and-rbac)
* [Clean up](#clean-up)



### Overview

The Kubernetes **Ingress** API is HTTP-only, hides advanced features behind vendor-specific annotations, and is one monolithic object every team shares. The **Gateway API** (GA since 2023) fixes these gaps with a role-oriented, portable set of resources:

* **GatewayClass** — the implementation (cluster-scoped). Calico Ingress Gateway provides `tigera-gateway-class`. *Owned by the Platform Engineer.*
* **Gateway** — a set of listeners (ports, protocols, TLS). *Owned by the Cluster Operator.*
* **HTTPRoute** — the routing rules (matches, filters, weighted `backendRefs`), attached to a Gateway via `parentRefs`. *Owned by the Application Developer.*

**Calico Ingress Gateway (CIG)** is Calico Enterprise's hardened distribution of the open-source **Envoy Gateway**, so everything you write is standard Gateway API and stays portable.

**How ingress is wired in this environment.** This is a self-managed kubeadm cluster on AWS, and Calico Ingress Gateway runs in **host-network mode**: the Envoy proxy is a **DaemonSet** with `hostNetwork: true`, so it binds ports **80 and 443 directly on the worker nodes' IPs** (the DaemonSet is pinned to the worker nodes). There is **no** `LoadBalancer`/`NodePort` Service and **no** BGP advertisement. Instead, an **AWS load balancer sits in front of the worker nodes and forwards port 443 to them**, where the host-network Envoy pods pick it up and route to the backends:

```
client ──> AWS load balancer (:443) ──> node IP (:443) ──> Envoy DaemonSet (hostNetwork) ──> backend pod
```

All `kubectl` and `curl` commands are run from the node you use to manage the cluster (control/bastion).

After finishing this lab, you should understand how to run Calico Ingress Gateway in host-network mode and use the Gateway API to expose and control north-south traffic.

______________________________________________________________________________________________________________________________________________________________________

### Enable Calico Ingress Gateway in host-network mode

> **Prerequisite — moved to the install lab.** The steps to enable Calico Ingress Gateway in host-network mode (create the `tigera-gateway` namespace, apply the host-network `EnvoyProxy`, enable the `GatewayAPI`, and bind `tigera-gateway-class`) are now part of the **[1. Install Calico Enterprise](../1.%20Install%20Calico%20Enterprise/#enable-calico-ingress-gateway-in-host-network-mode)** lab, alongside the base Calico Enterprise installation.
>
> Before continuing, make sure that section has been completed and the GatewayClass is `Accepted`:
>
> ```
> kubectl get gatewayclass
> ```


______________________________________________________________________________________________________________________________________________________________________


### Deploy the shared Gateway behind the AWS load balancer

In host-network mode a node can only have **one** Gateway binding 80/443, so we deploy a single shared Gateway in the `tigera-gateway` namespace and let every team attach routes to it. `allowedRoutes.namespaces.from: All` lets routes attach from any namespace.

#### Create a TLS certificate for the HTTPS listener

Create a TLS certificate for the HTTPS listener (the AWS load balancer forwards 443 to the nodes, so the 443 listener is the primary external entry point).

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=*.example.com" -addext "subjectAltName=DNS:*.example.com"
kubectl -n tigera-gateway create secret tls tigera-ingress-tls --cert=tls.crt --key=tls.key

```

#### Create the shared Gateway

Create the shared Gateway with an HTTP (80) and an HTTPS (443) listener.

```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tigera-ingress
  namespace: tigera-gateway
spec:
  gatewayClassName: tigera-gateway-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes: {namespaces: {from: All}}
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs: [{kind: Secret, name: tigera-ingress-tls}]
    allowedRoutes: {namespaces: {from: All}}
EOF

```

#### Verify the Envoy DaemonSet runs on the host network

Verify the Envoy DaemonSet is running on every **worker** node **on the host network** — the pod IP equals the node IP. (It runs on workers only because the `EnvoyProxy` created in the install lab pins the DaemonSet to the `node-role.kubernetes.io/worker` label.)

```
kubectl -n tigera-gateway get pods -o wide -l gateway.envoyproxy.io/owning-gateway-name=tigera-ingress

```

```
NAME                     READY   STATUS    IP           NODE
envoy-...-yyyyy          1/1     Running   10.0.1.30    worker1
envoy-...-zzzzz          1/1     Running   10.0.1.31    worker2
```

#### Confirm the Gateway is Programmed and store a node IP

Confirm the Gateway is `Programmed`, and store a **worker** node IP to test with. Because Envoy is on the host network and pinned to the workers, any worker node IP serves the Gateway; externally the AWS load balancer forwards 443 to these same worker nodes.

```
kubectl -n tigera-gateway get gateway tigera-ingress
NODE_IP=$(kubectl get nodes -l node-role.kubernetes.io/worker -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}') && echo "Node IP: $NODE_IP"

```

______________________________________________________________________________________________________________________________________________________________________

### Route HTTP traffic by host, path, and header

The core value of a Gateway is exposing many services behind one entry point, then routing by hostname, path, or header. HTTPRoutes live in the app namespace and attach to the shared Gateway via a cross-namespace `parentRefs` (allowed because the Gateway listener uses `from: All`).

#### Deploy two echo backends

Deploy two echo backends (each returns a fixed string so you can tell which one served the request).

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata: {name: ingress-demo}
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: web, namespace: ingress-demo}
spec:
  replicas: 1
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
      - name: web
        image: hashicorp/http-echo
        args: ["-text=Hello from WEB (v1)", "-listen=:5678"]
        ports: [{containerPort: 5678}]
---
apiVersion: v1
kind: Service
metadata: {name: web, namespace: ingress-demo}
spec:
  selector: {app: web}
  ports: [{port: 8080, targetPort: 5678}]
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: api, namespace: ingress-demo}
spec:
  replicas: 1
  selector: {matchLabels: {app: api}}
  template:
    metadata: {labels: {app: api}}
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args: ["-text=Hello from API", "-listen=:5678"]
        ports: [{containerPort: 5678}]
---
apiVersion: v1
kind: Service
metadata: {name: api, namespace: ingress-demo}
spec:
  selector: {app: api}
  ports: [{port: 8080, targetPort: 5678}]
EOF

```

#### Create the routes

Create the routes. `api.example.com` goes to `api`; `web.example.com` demonstrates **path** (`/api` → `api`), **header** (`x-env: canary` → `api`), and a default (`/` → `web`). Most specific match wins; unmatched requests return 404.

```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: api-host, namespace: ingress-demo}
spec:
  parentRefs: [{name: tigera-ingress, namespace: tigera-gateway}]
  hostnames: ["api.example.com"]
  rules:
  - backendRefs: [{name: api, port: 8080}]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: web-host, namespace: ingress-demo}
spec:
  parentRefs: [{name: tigera-ingress, namespace: tigera-gateway}]
  hostnames: ["web.example.com"]
  rules:
  - matches:
    - headers: [{name: x-env, value: canary}]
    backendRefs: [{name: api, port: 8080}]
  - matches:
    - path: {type: PathPrefix, value: /api}
    backendRefs: [{name: api, port: 8080}]
  - matches:
    - path: {type: PathPrefix, value: /}
    backendRefs: [{name: web, port: 8080}]
EOF

```

#### Test all three match types

Test all three match types by sending requests to a node IP with the right `Host` header.

```
curl -H "Host: api.example.com" http://$NODE_IP/          # host   -> API
curl -H "Host: web.example.com" http://$NODE_IP/          # host   -> WEB
curl -H "Host: web.example.com" http://$NODE_IP/api       # path   -> API
curl -H "Host: web.example.com" -H "x-env: canary" http://$NODE_IP/   # header -> API
curl -i -H "Host: nope.example.com" http://$NODE_IP/      # unmatched -> 404

```

______________________________________________________________________________________________________________________________________________________________________

### Split traffic for canary

Traffic splitting sends a percentage of traffic to a new version — progressive delivery — using **weighted `backendRefs`** on one `HTTPRoute`.

#### Deploy web v2 and split traffic 90/10

Deploy a v2 of the web app and split `web.example.com` 90/10 between v1 and v2.

```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata: {name: web-v2, namespace: ingress-demo}
spec:
  replicas: 1
  selector: {matchLabels: {app: web-v2}}
  template:
    metadata: {labels: {app: web-v2}}
    spec:
      containers:
      - name: web-v2
        image: hashicorp/http-echo
        args: ["-text=Hello from WEB (v2)", "-listen=:5678"]
        ports: [{containerPort: 5678}]
---
apiVersion: v1
kind: Service
metadata: {name: web-v2, namespace: ingress-demo}
spec:
  selector: {app: web-v2}
  ports: [{port: 8080, targetPort: 5678}]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: web-host, namespace: ingress-demo}
spec:
  parentRefs: [{name: tigera-ingress, namespace: tigera-gateway}]
  hostnames: ["web.example.com"]
  rules:
  - backendRefs:
    - {name: web, port: 8080, weight: 90}
    - {name: web-v2, port: 8080, weight: 10}
EOF

```

#### Send requests and observe the split

Send 20 requests and observe roughly 90/10. To shift the rollout, change the `weight` values; to roll back instantly, set v1 to 100 and v2 to 0 — no redeploy needed.

```
for i in $(seq 1 20); do curl -s -H "Host: web.example.com" http://$NODE_IP/ ; done | sort | uniq -c

```

```
     18 Hello from WEB (v1)
      2 Hello from WEB (v2)
```

______________________________________________________________________________________________________________________________________________________________________

### Terminate TLS at the Gateway

The shared Gateway's `https` listener terminates TLS with `mode: Terminate` using the `tigera-ingress-tls` Secret you created earlier — this is the listener the AWS load balancer forwards port 443 to. Terminating at the Gateway offloads encryption from your apps.

#### Attach a route to the HTTPS listener

Attach a route to the HTTPS listener for a secure hostname.

```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: secure-web, namespace: ingress-demo}
spec:
  parentRefs: [{name: tigera-ingress, namespace: tigera-gateway, sectionName: https}]
  hostnames: ["secure.example.com"]
  rules:
  - backendRefs: [{name: web, port: 8080}]
EOF

```

#### Test HTTPS on port 443

Test HTTPS against a node on port 443 (`-k` skips validation of the self-signed cert). This is exactly the path the AWS load balancer takes when it forwards client traffic on 443.

```
curl -k -H "Host: secure.example.com" https://$NODE_IP/

```

```
Hello from WEB (v1)
```

**Note:** Besides termination, Calico Ingress Gateway also supports **passthrough** (`mode: Passthrough` + `TLSRoute`, routed by SNI) and **re-encrypt** (terminate + a `BackendTLSPolicy` for the backend hop). In production you would typically use cert-manager to issue and rotate the certificate.

______________________________________________________________________________________________________________________________________________________________________

### Manipulate traffic with HTTPRoute filters

The Gateway can modify requests/responses at the edge — no backend changes — using **`HTTPRoute` filters**: header modifiers, redirects, and URL rewrites.

#### Apply header, rewrite, and redirect filters

Apply a route that adds/removes headers and rewrites the `/shop` prefix to `/`, plus a route that redirects HTTP to HTTPS for `secure.example.com`.

```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: web-filters, namespace: ingress-demo}
spec:
  parentRefs: [{name: tigera-ingress, namespace: tigera-gateway, sectionName: http}]
  hostnames: ["web.example.com"]
  rules:
  - matches:
    - path: {type: PathPrefix, value: /shop}
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add: [{name: x-added-by-gateway, value: cig}]
        remove: ["user-agent"]
    - type: URLRewrite
      urlRewrite:
        path: {type: ReplacePrefixMatch, replacePrefixMatch: /}
    backendRefs: [{name: web, port: 8080}]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: http-to-https, namespace: ingress-demo}
spec:
  parentRefs: [{name: tigera-ingress, namespace: tigera-gateway, sectionName: http}]
  hostnames: ["secure.example.com"]
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect: {scheme: https, statusCode: 301}
EOF

```

#### Confirm the HTTP-to-HTTPS redirect

Confirm the HTTP-to-HTTPS redirect (the header/rewrite filters take effect on requests to `web.example.com/shop/...`).

```
curl -i -H "Host: secure.example.com" http://$NODE_IP/

```

```
HTTP/1.1 301 Moved Permanently
location: https://secure.example.com:443/
```

**Note:** For a production-ready edge you can also add timeouts and retries (on the `HTTPRoute`), rate limiting (`BackendTrafficPolicy`, returns `429`), and authentication such as basic auth, JWT, or OIDC (`SecurityPolicy`, returns `401`) — all supported by Calico Ingress Gateway via Envoy Gateway policies.

______________________________________________________________________________________________________________________________________________________________________

### Self-service ingress with allowedRoutes and RBAC

Trust between a Gateway and a Route is **bidirectional**: the Route names its Gateway in `parentRefs`, and the Gateway must **allow** it via `allowedRoutes` (`from: Same` | `All` | `Selector`). Our shared Gateway uses `from: All`; here we tighten it to a **Selector** so only labeled namespaces may attach, and use **RBAC** so only the right people can create routes.

#### Restrict the HTTP listener to labeled namespaces

Label the existing app namespace so its routes keep working, then restrict the HTTP listener to namespaces carrying `ingress-access: "true"`.

```
kubectl label namespace ingress-demo ingress-access=true

kubectl patch gateway tigera-ingress -n tigera-gateway --type=merge -p '{"spec":{"listeners":[{"name":"http","protocol":"HTTP","port":80,"allowedRoutes":{"namespaces":{"from":"Selector","selector":{"matchLabels":{"ingress-access":"true"}}}}},{"name":"https","protocol":"HTTPS","port":443,"tls":{"mode":"Terminate","certificateRefs":[{"kind":"Secret","name":"tigera-ingress-tls"}]},"allowedRoutes":{"namespaces":{"from":"All"}}}]}}'

```

#### Create a route in an unlabeled namespace

As a developer, create a backend and a route in a new `dev` namespace (not yet labeled). The attachment is **rejected** because the namespace doesn't match the selector.

```
kubectl create namespace dev
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata: {name: store, namespace: dev}
spec:
  replicas: 1
  selector: {matchLabels: {app: store}}
  template:
    metadata: {labels: {app: store}}
    spec:
      containers:
      - name: store
        image: hashicorp/http-echo
        args: ["-text=Hello from the DEV store", "-listen=:5678"]
        ports: [{containerPort: 5678}]
---
apiVersion: v1
kind: Service
metadata: {name: store, namespace: dev}
spec:
  selector: {app: store}
  ports: [{port: 8080, targetPort: 5678}]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: {name: store, namespace: dev}
spec:
  parentRefs: [{name: tigera-ingress, namespace: tigera-gateway}]
  hostnames: ["store.example.com"]
  rules:
  - backendRefs: [{name: store, port: 8080}]
EOF

kubectl get httproute store -n dev -o jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].reason}{"\n"}'

```

```
NotAllowedByListeners
```

#### Label the dev namespace to accept the route

**The fix.** The Cluster Operator labels the `dev` namespace; the route flips to `Accepted` and now serves traffic.

```
kubectl label namespace dev ingress-access=true
kubectl get httproute store -n dev -o jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].reason}{"\n"}'
curl -H "Host: store.example.com" http://$NODE_IP/

```

```
Accepted
Hello from the DEV store
```

#### Enforce who can create routes with RBAC

Enforce *who* can create routes with RBAC, then prove the boundary. Bob can manage routes in `dev` but not Gateways in `tigera-gateway`.

```
kubectl create serviceaccount bob -n dev
kubectl create role route-developer -n dev --verb=get,list,create,update,delete --resource=httproutes.gateway.networking.k8s.io
kubectl create rolebinding bob-routes -n dev --role=route-developer --serviceaccount=dev:bob

kubectl auth can-i create httproutes -n dev            --as=system:serviceaccount:dev:bob   # -> yes
kubectl auth can-i create gateways   -n tigera-gateway --as=system:serviceaccount:dev:bob   # -> no

```

#### Restore the shared listener to from: All

Restore the shared listener to `from: All` for the rest of the environment.

```
kubectl patch gateway tigera-ingress -n tigera-gateway --type=merge -p '{"spec":{"listeners":[{"name":"http","protocol":"HTTP","port":80,"allowedRoutes":{"namespaces":{"from":"All"}}},{"name":"https","protocol":"HTTPS","port":443,"tls":{"mode":"Terminate","certificateRefs":[{"kind":"Secret","name":"tigera-ingress-tls"}]},"allowedRoutes":{"namespaces":{"from":"All"}}}]}}'

```

**Note:** Don't confuse `allowedRoutes` (Gateway↔Route trust, shown here) with `ReferenceGrant`. `allowedRoutes` governs which namespaces' routes may attach to a Gateway. A `ReferenceGrant` is a separate concern — it is only needed when a Route's `backendRefs` points to a Service in **another namespace**, or when a Gateway references a Secret in another namespace. Here the `dev` route's backend lives in `dev` (same namespace), so no `ReferenceGrant` is required.

______________________________________________________________________________________________________________________________________________________________________

### Clean up

```
kubectl delete ns ingress-demo dev
kubectl delete gateway tigera-ingress -n tigera-gateway
kubectl delete secret tigera-ingress-tls -n tigera-gateway
rm -f tls.crt tls.key

```

*(Optional)* Fully remove host-network Calico Ingress Gateway:

```
kubectl delete gatewayapi tigera-secure
kubectl delete envoyproxy envoy-hostnet -n tigera-gateway
kubectl delete namespace tigera-gateway

```

> **Congratulations! You have completed `6. Calico Ingress Gateway` lab.**
