# Kubernetes Gateway API

## Prerequisites

Create cluster:

```bash
task cluster-create
```

## Examples

### Ingress NGINX

Provision Ingress NGINX:

```bash
task cluster-ingress-setup
```

Create podinfo application:

```bash
task app-podinfo-create
```

Check podinfo application:

```bash
task app-podinfo-check
```

Traffic flow:

```
  macOS Terminal
  │
  │  curl -H 'Host: podinfo.example.com' http://localhost
  │    (or: curl -kiv https://podinfo.example.com)
  │
  ▼
  localhost:80 (or :443)
  │  Kind extraPortMapping · maps macOS localhost ports to the Kind node container
  ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Docker  (kind network · 172.18.0.0/16)                                     │
│                                                                            │
│  home-lab-control-plane  (KIND node · Docker container)                    │
│  · containerPort :80/:443 bound directly via hostPort on the ingress pod   │
│  │                                                                         │
│ ┌──────────────────────────────────────────────────────────────────────┐   │
│ │ Kubernetes cluster                                                   │   │
│ │                                                                      │   │
│ │  ns: ingress-nginx                                                   │   │
│ │  ingress-nginx-controller pod  (hostPort: 80/443)                    │   │
│ │  · IngressClass: nginx                                               │   │
│ │  · terminates TLS (self-signed cert)                                 │   │
│ │  · matches Host header against Ingress rules                         │   │
│ │  │  Ingress: podinfo.example.com → podinfo svc :9898                 │   │
│ │  ▼                                                                   │   │
│ │  ns: default                                                         │   │
│ │  podinfo svc :9898  ──►  podinfo pod  (ghcr.io/stefanprodan/podinfo) │   │
│ │                          · responds with request details,            │   │
│ │                            environment info, and metrics             │   │
│ └──────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────┘
```

### Gateway API

#### Kubernetes Cloud Provider for KIND

Provision cloud provider:

```bash
task cloud-provider-setup
```

Create kind gateway:

```bash
task kind-gateway-setup
```

Create echo application:

```bash
task app-echo-kind-create
```

Check echo application:

```bash
task app-echo-kind-check
```

Traffic flow:

```
  macOS Terminal
  │
  │  curl -H 'Host: echo.example.gateway.kind.com' http://localhost:8080
  │
  ▼
  localhost:8080
  │  Docker port mapping · bridges macOS localhost into the kind network
  ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Docker  (kind network · 172.18.0.0/16)                                     │
│                                                                            │
│  kind-gateway-proxy-kind  (socat container)                                │
│  · forwards raw TCP from :80 into the kind network                         │
│  :80  ──────────────────────────────────────────────►  172.18.0.x:80       │
│                                                                  │         │
│  kindccm-xxxxxxxxxx  (cloud-provider-kind · Envoy proxy)         │         │
│  · Gateway implementation for KIND; assigned external IP         │         │
│  · matches Host header against HTTPRoute rules                   │         │
│  172.18.0.x:80  ◄────────────────────────────────────────────────┘         │
│  │  routes directly to pod IP via kind network                             │
│  ▼                                                                         │
│ ┌──────────────────────────────────────────────────────────────────────┐   │
│ │ Kubernetes cluster                                                   │   │
│ │                                                                      │   │
│ │  ns: gateway-infra-kind                                              │   │
│ │  GatewayClass: cloud-provider-kind                                   │   │
│ │  · hostname: *.example.gateway.kind.com                              │   │
│ │  │  HTTPRoute: echo.example.gateway.kind.com → echo svc :3000        │   │
│ │  ▼                                                                   │   │
│ │  ns: demo                                                            │   │
│ │  echo svc :3000  ──►  echo pod  (echo-basic)                         │   │
│ │                       · responds with request details (path,         │   │
│ │                         headers, pod name, namespace)                │   │
│ └──────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────┘
```

#### Istio Gateway API

Provision cloud provider for load balancer support:

```bash
task cloud-provider-setup
```

Install Istio:

```bash
task istio-setup
```

Create Istio Gateway:

```bash
task istio-gateway-setup
```

Create echo application with Istio sidecar injection:

```bash
task app-echo-istio-create
```

Check echo application through Istio Gateway:

```bash
task app-echo-istio-check
```

Traffic flow:

```
  macOS Terminal
  │
  │  curl -H 'Host: echo.example.gateway.istio.com' http://localhost:8081
  │
  ▼
  localhost:8081
  │  Docker port mapping · bridges macOS localhost into the kind network
  ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Docker  (kind network · 172.18.0.0/16)                                     │
│                                                                            │
│  kind-gateway-proxy-istio  (socat container)                               │
│  · forwards raw TCP from :80 into the kind network                         │
│  :80  ──────────────────────────────────────────────►  172.18.0.3:80       │
│                                                                  │         │
│  kindccm-xxxxxxxxxx  (cloud-provider-kind · Envoy proxy)         │         │
│  · LoadBalancer implementation for KIND; assigned external IP    │         │
│  172.18.0.3:80  ◄────────────────────────────────────────────────┘         │
│  │  routes via nodePort                                                    │
│  ▼                                                                         │
│  172.18.0.2:31112                                                          │
│  │  home-lab-control-plane  (KIND node · 172.18.0.2)                       │
│  │  kube-proxy forwards nodePort traffic to the gateway pod                │
│  ▼                                                                         │
│ ┌──────────────────────────────────────────────────────────────────────┐   │
│ │ Kubernetes cluster                                                   │   │
│ │                                                                      │   │
│ │  ns: gateway-infra-istio                                             │   │
│ │  gateway-istio pod  (Istio Envoy gateway · 10.244.0.12:80)           │   │
│ │  · GatewayClass: istio                                               │   │
│ │  · matches Host header against HTTPRoute rules                       │   │
│ │  │  HTTPRoute: echo.example.gateway.istio.com → echo svc :3000       │   │
│ │  ▼                                                                   │   │
│ │  ns: demo                                                            │   │
│ │  echo svc :3000  ──►  echo pod  (10.244.0.13:3000)                   │   │
│ │                       · responds with request details (path,         │   │
│ │                         headers, pod name, namespace)                │   │
│ └──────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────┘
```

## Cleanup

```bash
task cloud-provider-delete
task cluster-delete
task docker-clean
```

## Links

* [Kubernetes Ingress NGINX](https://github.com/kubernetes/ingress-nginx)
* [Istio - Kubernetes Gateway API](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/)
* [Istio - Kubernetes Gateway API on KIND](https://istio.io/latest/docs/setup/platform-setup/kind/)
* [Configure Istio ingress with the Kubernetes Gateway API for Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/istio-gateway-api)
* [Experimenting with Gateway API using kind](https://kubernetes.io/blog/2026/01/28/experimenting-gateway-api-with-kind/)
* [Kubernetes Cloud Provider for KIND](https://github.com/kubernetes-sigs/cloud-provider-kind)
* [Building a Modern Kubernetes Ingress with Istio Gateway API: A Complete Guide](https://medium.com/@guptaaayush8/building-a-modern-kubernetes-ingress-with-istio-gateway-api-a-complete-guide-1a8897629f55)
* [Setting up Istio Ingress With Kubernetes Gateway API](https://devopscube.com/istio-ingress-kubernetes-gateway-api/)
* [Configure helloworld using the Kubernetes Gateway API](https://github.com/istio/istio/blob/master/samples/helloworld/gateway-api/README.md)
* [Kubernetes Gateway API + Istio For North/South Traffic](https://www.cloudnativedeepdive.com/kubernetes-gateway-api-istio-for-north-south-traffic/)
* [How to Use Kubernetes Gateway API with Istio](https://oneuptime.com/blog/post/2026-02-24-how-to-use-kubernetes-gateway-api-with-istio/view)
