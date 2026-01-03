---
layout: post
title: "Kubernetes in a Hurry: From kube-proxy to ServiceMesh(Q&A Format)"
categories: Learn in a hurry
author:
- Yuva Peavler
---

If you need networking fundamentals primer before diving into container networking and kubernetes, check out [Part 1 of this series: Networking in a hurry](https://ypeavler.github.io/blog/learn/in/a/hurry/2026/01/01/networking-in-a-hurry.html)


**Test Your Knowledge**
Ready to test your understanding? Take the **[Networking Quiz](https://ypeavler.github.io/blog/2026/01/01/networking-basics-quiz.html)** — 34 questions covering everything from ARP to service mesh.

---

## Part 1: Kubernetes Networking

<a id="what-are-the-three-fundamental-requirements-of-kubernetes-networking"></a>

#### Q: What are the three fundamental requirements of Kubernetes networking?

Kubernetes has a simple networking model with three fundamental requirements:

1. **Every pod gets its own IP address** — No NAT between pods
2. **Pods can communicate with all other pods** — Without NAT, across all nodes
3. **Agents on a node can communicate with all pods** — Without NAT

This model is implemented by **CNI (Container Network Interface)** plugins.

---

<a id="what-is-kube-proxy-and-what-are-its-modes"></a>

#### Q: What is kube-proxy and what are its modes?

kube-proxy is a Kubernetes component that maintains network rules on each node, enabling Service abstraction. It has three modes:

- **iptables mode** (default): Creates iptables rules for Service IP → Pod IP mapping. Fast and efficient, but rules can become large with many services.
- **ipvs mode**: Uses Linux IPVS (IP Virtual Server) for load balancing. Better performance and scalability than iptables for large clusters.
- **userspace mode** (deprecated): Proxy runs in userspace. Slower and rarely used.

---

<a id="can-you-show-an-example-of-how-kube-proxy-redirects-service-traffic"></a>

#### Q: Can you show an example of how kube-proxy redirects Service traffic?

```bash
# Service 10.96.0.100:80 → Pods 10.244.1.5:8080, 10.244.2.7:8080
$ iptables -t nat -L KUBE-SERVICES
Chain KUBE-SERVICES
KUBE-SVC-XXX  tcp  --  anywhere  10.96.0.100  tcp dpt:80

$ iptables -t nat -L KUBE-SVC-XXX
Chain KUBE-SVC-XXX
KUBE-SEP-AAA  all  --  anywhere  anywhere  statistic mode random probability 0.5
KUBE-SEP-BBB  all  --  anywhere  anywhere  # remaining 50%
```

---

<a id="what-is-cni"></a>

#### Q: What is CNI?

*CNI (Container Network Interface)* is the standard for Kubernetes networking plugins. When a pod starts, it's like a new apartment being built. The CNI plugin is like the city planning department that assigns the new apartment an address (IP address), connects it to the street (creates veth pair), and gives the resident a mailbox (network namespace). The pod can now send and receive letters (packets) just like any other apartment in the city!

<div class="mermaid">
sequenceDiagram
    autonumber
    participant Kubelet as kubelet
    participant CNI as CNI Plugin
    participant Pod as Pod

    Kubelet->>CNI: ADD: Create network

    Note over CNI: Create namespace<br/>Create veth pair<br/>Assign IP<br/>Configure routes

    CNI->>Pod: Network ready<br/>IP: 10.244.1.15

    CNI->>Kubelet: Success

    Note over Kubelet,Pod: Pod ready
</div>

---

<a id="what-are-the-cni-plugin-responsibilities"></a>

#### Q: What are the CNI plugin responsibilities?

1. Create network namespace for the pod
2. Create veth pair (one end in pod, one in host)
3. Assign IP address from IPAM (IP Address Management) — IPAM allocates IPs from a configured CIDR range
4. Configure routes so pod can reach other pods and services
5. Set up overlay network (if needed) for cross-node communication

---

<a id="what-are-popular-cni-plugins"></a>

#### Q: What are popular CNI plugins?

| CNI Plugin | Approach | Overlay Protocol | Key Features |
   |------------|----------|------------------|--------------|
   | **Cilium** | eBPF-based routing | Geneve, VXLAN, or native | Network policies, observability, service mesh integration |
   | **Calico** | BGP routing | VXLAN or native | Network policies, BGP integration |
   | **Flannel** | Simple overlay | VXLAN | Simple, minimal configuration |
   | **Weave** | Mesh overlay | Custom (sleeve/fastdp) | Automatic mesh networking |

For enterprise and multi-tenant Kubernetes deployments, **Cilium** and **Calico** are often preferred due to their robust network policy enforcement, performance benefits (eBPF for Cilium), and integration with BGP for native routing, which are critical for security and scalability.

---


<a id="what-is-cilium-and-what-makes-it-special"></a>

#### Q: What makes cilium special?

Cilium uses eBPF (extended Berkeley Packet Filter) for high-performance networking:

**Key features:**

- **eBPF-based routing**: Faster than iptables, no kernel bypass needed
- **Network policies**: Enforced at the kernel level using eBPF programs
- **Observability**: Built-in metrics and tracing
- **Service mesh integration**: Can replace kube-proxy

**Routing modes:**

- **Geneve overlay (default)** — Uses Geneve encapsulation for cross-node pod communication. Default choice when overlay is needed. Provides rich metadata in TLV options for advanced network policy enforcement.
- **VXLAN overlay** — Alternative to Geneve for compatibility with environments that don't support Geneve. Similar functionality but without TLV extensibility.
- **Native routing** — No overlay encapsulation. Routes pod IPs directly through the underlying network. Requires routable pod IPs and BGP or static routes.
- **BGP routing** — Uses BGP to advertise pod CIDR routes to network infrastructure. Enables native routing with dynamic route distribution. Works with routers, cloud provider route tables, and other BGP-speaking devices.

---

<a id="what-is-ipam-ip-address-management"></a>

#### Q: What is IPAM (IP Address Management)?

IPAM is the component of CNI plugins that manages IP address allocation. Each CNI plugin includes an IPAM plugin (or uses a standalone one like `host-local`) that:

- Allocates IP addresses from a configured CIDR range (e.g., 10.244.0.0/16)
- Tracks which IPs are assigned to which pods
- Releases IPs when pods are deleted
- Prevents IP conflicts by ensuring each pod gets a unique IP

---

<a id="how-do-pods-on-the-same-node-communicate"></a>

#### Q: How do pods on the same node communicate?

When two pods are on the same node, they communicate through the node's bridge. When two pods are on the same node, it's like two apartments in the same building. You write a letter to your neighbor, drop it in the building's mailroom (bridge), and the mailroom immediately delivers it to your neighbor's apartment. No need for the postal service—it's all handled within the building!

<div class="mermaid">
flowchart TB
    subgraph Node["Kubernetes Node"]
        subgraph Pod1["Pod 1 10.244.1.5"]
            APP1["App Container"]
        end

        subgraph Pod2["Pod 2 10.244.1.6"]
            APP2["App Container"]
        end

        BRIDGE["CNI Bridge (cni0, cali0, etc.) 10.244.1.1"]
    end

    Pod1 --> BRIDGE
    Pod2 --> BRIDGE
    BRIDGE <--> Pod1
    BRIDGE <--> Pod2
</div>

---

<a id="how-do-pods-on-different-nodes-communicate"></a>

#### Q: How do pods on different nodes communicate?

When pods are on different nodes, the CNI plugin uses an overlay network (VXLAN or Geneve). It's like sending a letter from one building to another across town. You write your letter (original packet) and put it in an inner envelope addressed to your friend's apartment (destination pod IP). The building's mailroom (CNI plugin) puts that inner envelope inside an outer envelope addressed to the destination building (node IP). The postal service (underlay network) delivers the outer envelope to the destination building, where the mailroom there opens it and delivers the inner envelope to your friend's apartment. Your friend never sees the outer envelope—they just receive your letter!

<div class="mermaid">
sequenceDiagram
    autonumber
    participant Pod1 as Pod1
    participant CNI1 as CNI1
    participant Overlay as Overlay
    participant CNI2 as CNI2
    participant Pod2 as Pod2

    Pod1->>CNI1: Packet to Pod2

    Note over CNI1: Route lookup<br/>Encapsulate in overlay

    Note over CNI1,CNI2: OVERLAY ENCAPSULATION
    CNI1->>Overlay: Outer: Node1 → Node2<br/>Inner: Pod1 → Pod2
    Overlay->>CNI2: Routed

    Note over CNI2: Decapsulate<br/>Extract packet

    CNI2->>Pod2: Packet delivered

    Note over Pod1,Pod2: Pods on same network
</div>

---

<a id="why-do-we-need-kubernetes-services"></a>

#### Q: Why do we need Kubernetes Services?

Pods are ephemeral — they can be created, destroyed, and moved. *Services* provide a stable endpoint for pods. Instead of tracking individual pod IPs (which change constantly), applications use a Service IP that remains constant. kube-proxy maintains the mapping between Service IPs and pod IPs, automatically updating it as pods are created or destroyed.

---

<a id="what-are-the-different-service-types"></a>

#### Q: What are the different Service types?

| Service Type | Use Case | How It Works |
|--------------|----------|--------------|
| **ClusterIP** | Internal communication | Virtual IP (10.96.0.0/12) that kube-proxy routes to pod IPs |
| **NodePort** | External access via node IP | Opens a port (30000-32767) on all nodes, routes to ClusterIP |
| **LoadBalancer** | Cloud provider integration | Creates external load balancer, routes to NodePort |
| **ExternalName** | External service alias | DNS CNAME to external service |
| **Headless** | Direct pod access | No ClusterIP, DNS returns pod IPs directly |

---

<a id="how-does-clusterip-work"></a>

#### Q: How does ClusterIP work?

1. When a Service is created, Kubernetes assigns it a ClusterIP (a virtual IP, e.g., 10.96.0.100) from the cluster's service CIDR (typically 10.96.0.0/12)
2. kube-proxy on each node creates iptables rules that map the ClusterIP → Pod IPs (based on Endpoints/EndpointSlices)
3. When traffic arrives at a node destined for the ClusterIP, kube-proxy's iptables rules perform DNAT (Destination NAT), rewriting the destination IP from the ClusterIP to a selected pod IP (load balanced across available pods)
4. If the selected pod is on a different node, the overlay network (VXLAN/Geneve) handles routing the packet to the destination pod's node

<div class="mermaid">
flowchart TB
    subgraph Service["Service: backend ClusterIP: 10.96.0.100:80"]
        SVC_IP["ClusterIP 10.96.0.100"]
    end

    subgraph KubeProxy["kube-proxy (iptables/IPVS)"]
        RULES["iptables DNAT rules: 10.96.0.100:80 → 50% → 10.244.1.5:8080 50% → 10.244.2.10:8080 (Pod 3 not ready)"]
    end

    subgraph Pods["Backend Pods"]
        Pod1["Pod 1 10.244.1.5:8080"]
        Pod2["Pod 2 10.244.2.10:8080"]
        Pod3["Pod 3 10.244.3.15:8080 (not ready)"]
    end

    SVC_IP --> KubeProxy
    KubeProxy --> Pod1
    KubeProxy --> Pod2
</div>

---

<a id="why-do-we-need-endpoints"></a>

#### Q: Why do we need Endpoints?

Endpoints solve a fundamental problem in Kubernetes: **Services have stable IPs, but Pods have ephemeral IPs that change constantly.**

**The problem:**

- A Service provides a stable virtual IP (e.g., 10.96.0.100) that applications can use
- But pods are ephemeral—they get new IPs every time they start, restart, or move to a different node
- When a pod is created, destroyed, or scaled, its IP changes
- kube-proxy needs to know which actual pod IPs to route traffic to

**Without Endpoints:**

- kube-proxy would have to constantly query the Kubernetes API to find pod IPs
- This would be inefficient and slow
- There would be no single source of truth for "which pods belong to this Service?"

**With Endpoints:**

- Kubernetes automatically creates and maintains an Endpoints resource for each Service
- The Endpoints resource lists all current pod IPs that match the Service selector
- kube-proxy watches the Endpoints resource (not individual pods)
- When pods change, the Endpoints resource is updated automatically
- kube-proxy gets notified of changes and updates its routing rules (iptables/IPVS)

**The bridge:**

Endpoints bridge the gap between:

- **Services** (stable, logical abstraction: "the backend service")
- **Pods** (ephemeral, physical reality: "these 5 pods with IPs 10.244.1.5, 10.244.2.10, ...")

Think of it like a phone book: The Service is like a business name ("Acme Corp"), and Endpoints is the phone book that lists all current phone numbers (pod IPs) for that business. When employees (pods) come and go, the phone book (Endpoints) is updated automatically.

<div class="mermaid">
flowchart LR
    subgraph Service["Service backend ClusterIP: 10.96.0.100 (Stable, never changes)"]
        SVC["Service Resource"]
    end

    subgraph Endpoints["Endpoints Resource (The Bridge)"]
        EP["Endpoints 10.244.1.5:8080 10.244.2.10:8080 10.244.3.15:8080 (Auto-updated)"]
    end

    subgraph Pods["Pods (Ephemeral IPs)"]
        P1["Pod 1 10.244.1.5:8080"]
        P2["Pod 2 10.244.2.10:8080"]
        P3["Pod 3 10.244.3.15:8080"]
    end

    subgraph KubeProxy["kube-proxy (watches Endpoints)"]
        KP["iptables rules based on Endpoints"]
    end

    SVC -->|"Service selector matches pods"| EP
    EP -->|"kube-proxy watches for changes"| KP
    P1 -.->|"Pod IPs change on restart/scale"| EP
    P2 -.->|"Pod IPs change on restart/scale"| EP
    P3 -.->|"Pod IPs change on restart/scale"| EP
    KP -->|"Routes traffic"| P1
    KP -->|"Routes traffic"| P2
    KP -->|"Routes traffic"| P3
</div>

---

<a id="what-are-endpoints-and-endpointslices"></a>

#### Q: What are Endpoints and EndpointSlices?

Endpoints (and the newer EndpointSlices) are Kubernetes resources that track which pod IPs belong to a Service. They bridge the gap between Services (which have stable IPs) and Pods (which have ephemeral IPs).

**Endpoints:**

- A Kubernetes resource that lists all pod IPs and ports for a Service
- Automatically created and updated by the Kubernetes control plane as pods matching the Service selector are created/destroyed
- Each Service has one Endpoints resource (or none if no pods match)
- Contains all pod IPs in a single resource, which can become large with many pods

**Example Endpoints resource:**
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: backend  # Matches Service name
subsets:
- addresses:
  - ip: 10.244.1.5
    nodeName: node-1
  - ip: 10.244.2.10
    nodeName: node-2
  ports:
  - port: 8080
    protocol: TCP
```

**EndpointSlices:**

- A newer, more scalable alternative to Endpoints (introduced in Kubernetes 1.16, GA in 1.21)
- Splits endpoints across multiple slice resources (up to 100 endpoints per slice)
- Reduces the size of individual resources, improving performance in large clusters
- Provides better scalability: a Service with 1000 pods creates ~10 EndpointSlices instead of 1 large Endpoints resource
- Includes additional metadata like topology hints (which zone/node pods are in)

**Example EndpointSlice:**
```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: backend-abc123
  labels:
    kubernetes.io/service-name: backend
addressType: IPv4
ports:
- name: http
  port: 8080
  protocol: TCP
endpoints:
- addresses:
  - "10.244.1.5"
  conditions:
    ready: true
  nodeName: node-1
  zone: us-west-1a
- addresses:
  - "10.244.2.10"
  conditions:
    ready: true
  nodeName: node-2
  zone: us-west-1b
```

**How kube-proxy uses them:**

- kube-proxy watches Endpoints or EndpointSlices (EndpointSlices preferred in modern clusters)
- When endpoints change (pod created/destroyed), kube-proxy updates iptables or IPVS rules
- Rules map Service IP → Pod IPs, enabling load balancing across pods
- The watch mechanism ensures rules stay synchronized with actual pod state

**Why EndpointSlices matter:**

- **Performance**: Smaller resources mean faster API server processing and less network traffic
- **Scalability**: Can handle services with thousands of pods without creating massive single resources
- **Topology awareness**: Includes zone/node information for better routing decisions
- **Future-proof**: Foundation for advanced features like topology-aware routing

---

<a id="how-does-service-discovery-work-with-dns"></a>

#### Q: How does service discovery work with DNS?

Kubernetes provides DNS for services via CoreDNS. Applications can use service names (e.g., `backend.default.svc.cluster.local`) instead of IP addresses. CoreDNS resolves service names to Service IPs, which kube-proxy then routes to pod IPs.

<div class="mermaid">
sequenceDiagram
    autonumber
    participant App as App
    participant DNS as CoreDNS
    participant KubeProxy as kube-proxy
    participant Backend as Backend

    App->>DNS: Query: backend.default.svc

    Note over DNS: Lookup:<br/>backend → 10.96.0.100

    DNS->>App: A: 10.96.0.100

    App->>KubeProxy: GET 10.96.0.100:80/api

    Note over KubeProxy: DNAT:<br/>10.96.0.100 → 10.244.2.10

    KubeProxy->>Backend: GET /api

    Backend->>KubeProxy: 200 OK
    KubeProxy->>App: 200 OK
</div>

---

<a id="what-is-coredns"></a>

#### Q: What is CoreDNS?

CoreDNS is the default DNS server in Kubernetes. It:

- Runs as a Deployment in the `kube-system` namespace
- Watches Kubernetes Services and Endpoints
- Automatically creates DNS records for all Services
- Resolves service names to Service IPs (ClusterIP)
- Supports custom DNS entries via ConfigMaps

When a pod queries `backend.default.svc.cluster.local`, CoreDNS returns the Service IP (e.g., 10.96.0.100), which kube-proxy then routes to an actual pod IP.

---

<a id="what-is-the-dns-naming-format"></a>

#### Q: What is the DNS naming format?

- Format: `<service>.<namespace>.svc.cluster.local`
- Short form: `<service>.<namespace>` or just `<service>` (same namespace)
- Example: `backend.default.svc.cluster.local` → `10.96.0.100`

---

<a id="can-you-walk-through-a-dns-service-discovery-flow"></a>

#### Q: Can you walk through a DNS service discovery flow?

The sequence diagram above shows the complete DNS service discovery flow. Applications query DNS using service names, CoreDNS resolves to ClusterIPs, and kube-proxy routes to pod IPs via overlay networks when needed.

---

<a id="what-is-ingress-and-how-does-it-relate-to-services"></a>

#### Q: What is Ingress and how does it relate to Services?

Ingress provides HTTP/HTTPS routing from outside the cluster to Services. Unlike Services (which provide internal cluster networking), Ingress handles external access.

- **Ingress Controller**: A reverse proxy (e.g., NGINX, Traefik, Envoy) that runs in the cluster and implements Ingress rules
- **Ingress Resource**: Defines routing rules (host, path → Service)
- **Flow**: External request → Ingress Controller → Service → Pod

Ingress works *on top of* Services—it routes external traffic to the appropriate Service, which then routes to pods. For advanced routing (canary, A/B testing, mTLS), service mesh is often used instead of or alongside Ingress.

<div class="mermaid">
sequenceDiagram
    autonumber
    participant User as User
    participant LB as Load Balancer
    participant IC as Ingress
    participant SVC as Service
    participant Pod as Backend

    User->>LB: GET https://api.example.com/api/data
    LB->>IC: Request<br/>Host: api.example.com

    Note over IC: Ingress:<br/>api.example.com → backend

    IC->>SVC: GET /api/data

    Note over SVC: DNAT:<br/>10.96.0.100 → 10.244.2.10

    SVC->>Pod: GET /api/data

    Pod->>SVC: 200 OK
    SVC->>IC: 200 OK
    IC->>LB: 200 OK
    LB->>User: 200 OK
</div>

---

<a id="what-are-network-policies"></a>

#### Q: What are Network Policies?

Network Policies allow you to control traffic between pods using label selectors. They act as pod-level firewalls, allowing or denying traffic based on source pod labels, destination pod labels, and ports. Network Policies are enforced by CNI plugins (Cilium, Calico) at the kernel level, before packets reach the pod.

<div class="mermaid">
flowchart TB
    subgraph Allowed["Allowed Traffic"]
        direction LR
        F1["frontend pod"]
        B1["backend pod"]
        F1 -->|"Port 8080"| B1
    end

    subgraph Blocked["Blocked Traffic"]
        direction LR
        F2["frontend pod"]
        B2["backend pod"]
        OTHER["other pod"]
        F2 -.->|"Port 8081"| B2
        OTHER -.->|"Any port"| B2
    end
</div>

---

<a id="can-you-show-an-example-networkpolicy"></a>

#### Q: Can you show an example NetworkPolicy?

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

This policy says: "Only pods labeled `app: frontend` can talk to pods labeled `app: backend` on port 8080."

---

<a id="how-are-network-policies-implemented"></a>

#### Q: How are Network Policies implemented?

- **Cilium**: Uses eBPF and Geneve TLV options for policy enforcement
- **Calico**: Uses iptables rules and BGP for policy distribution
- **Flannel**: Does not support Network Policies (needs Calico or Cilium)

**NetworkPolicy vs service mesh authorization (multi-tenant isolation)**

| Aspect | NetworkPolicy (CNI) | Service mesh auth (sidecar) |
|--------|---------------------|-----------------------------|
| Scope | L3/L4 (IP, port) | L7 (HTTP/gRPC) + identity |
| Enforcer | CNI dataplane (node) | Sidecar proxy (pod) |
| Best for | Blast-radius limits between namespaces/tenants; default-deny | App-level allow/deny, mTLS, per-route rules |
| Debug with | `kubectl get networkpolicies -A`, `cilium monitor`/`calicoctl` | Sidecar logs, `AuthorizationPolicy`/`ServerAuthorization` |



<a id="what-is-the-kubernetes-networking-stack"></a>

#### Q: What is the Kubernetes networking stack?

The Kubernetes networking stack is built in layers, with each layer providing specific functionality:

```
┌─────────────────────────────────────┐
│  Service Mesh (L7)                  │  ← Identity, observability, traffic mgmt
│  (Istio, Linkerd)                   │
├─────────────────────────────────────┤
│  Services (kube-proxy)              │  ← Service discovery, load balancing
│  ClusterIP, NodePort, LoadBalancer  │
├─────────────────────────────────────┤
│  CNI Plugin (L2/L3)                 │  ← Pod networking, overlay networks
│  (Cilium, Calico, Flannel)          │
├─────────────────────────────────────┤
│  Linux Networking Primitives        │  ← Network namespaces, veth, bridges
└─────────────────────────────────────┘
```

**Key takeaways:**

- CNI plugins provide pod networking using native or overlay networks (VXLAN/Geneve)
- Services abstract pod IPs using virtual IPs and kube-proxy
- Network Policies control traffic between pods
- Service mesh adds Layer 7 capabilities on top of everything

---


## Part 2: Cilium Special

<a id="what-are-the-requirements-for-cilium-native-routing-mode"></a>

#### Q: What are the requirements for Cilium native routing mode?

Native routing mode eliminates overlay encapsulation, routing pod IPs directly through the underlying network. This requires:

1. **Routable pod IPs**: Pod IPs (from the pod CIDR, e.g., 10.244.0.0/16) must be routable in your network infrastructure. The underlying network (routers, switches, cloud provider networking) must know how to route traffic to pod IPs.

2. **BGP or static routes**: You need either:
   - **BGP**: Cilium can use BGP to advertise pod CIDR routes to your network infrastructure (routers, cloud provider route tables)
   - **Static routes**: Manually configure routes in your network infrastructure pointing pod CIDRs to Kubernetes nodes

3. **No IP conflicts**: Pod IPs must not conflict with existing IPs in your network.

**When to use native routing:**

- You have control over network infrastructure (on-premises, custom cloud networking)
- You want maximum performance (no encapsulation overhead)
- Your network supports BGP or you can configure static routes
- Pod IPs can be made routable in your network

**When to use overlay (Geneve/VXLAN):**

- Cloud provider environments where pod IPs aren't routable
- You want simplicity (no BGP/static route configuration)
- Network infrastructure doesn't support routing pod CIDRs
- You need the rich metadata capabilities of Geneve TLV options

---

<a id="how-does-cilium-use-bgp"></a>

#### Q: How does Cilium use BGP?

Cilium supports BGP for native routing mode, allowing it to advertise pod CIDR routes to network infrastructure without using overlay encapsulation. Each Cilium node runs a BGP daemon that advertises its pod CIDR to BGP peers (routers, cloud provider route tables), enabling the network infrastructure to learn pod IP routes and route traffic directly to pods without encapsulation overhead.

<div class="mermaid">
flowchart TB
    subgraph K8s["Kubernetes Cluster"]
        subgraph Node1["Node 1 Cilium Agent"]
            Pod1["Pod: 10.244.1.5"]
            Pod2["Pod: 10.244.1.6"]
            BGP1["Cilium BGP Daemon Pod CIDR: 10.244.1.0/24"]
        end

        subgraph Node2["Node 2 Cilium Agent"]
            Pod3["Pod: 10.244.2.10"]
            Pod4["Pod: 10.244.2.11"]
            BGP2["Cilium BGP Daemon Pod CIDR: 10.244.2.0/24"]
        end

        subgraph Node3["Node 3 Cilium Agent"]
            Pod5["Pod: 10.244.3.15"]
            BGP3["Cilium BGP Daemon Pod CIDR: 10.244.3.0/24"]
        end
    end

    subgraph Network["Network Infrastructure"]
        Router["BGP Router or Route Reflector"]
        CloudRT["Cloud Route Table (AWS/Azure/GCP)"]
    end

    BGP1 -->|"BGP Advertisement: 10.244.1.0/24 → Node1"| Router
    BGP2 -->|"BGP Advertisement: 10.244.2.0/24 → Node2"| Router
    BGP3 -->|"BGP Advertisement: 10.244.3.0/24 → Node3"| Router

    Router --> CloudRT

    Router -.->|"Learns routes: 10.244.1.0/24 → Node1 10.244.2.0/24 → Node2 10.244.3.0/24 → Node3"| Router
</div>

**Use cases for Cilium BGP:**

- **On-premises deployments**: Advertise pod routes to physical routers/switches
- **Cloud environments**: Integrate with cloud provider route tables (AWS Route Tables, Azure Route Tables, GCP Routes)
- **Hybrid cloud**: Connect on-premises and cloud networks via BGP
- **Large-scale clusters**: Native routing performs better than overlay at scale
- **Integration with existing BGP infrastructure**: Works with existing network equipment

**BGP vs overlay in Cilium:**

- **BGP (native routing)**: Better performance, no encapsulation overhead, requires BGP-capable infrastructure
- **Overlay (Geneve/VXLAN)**: Works everywhere, simpler setup, adds encapsulation overhead

**Configuration example:**
Cilium can be configured to use BGP by enabling the BGP control plane and specifying BGP peers (routers or route reflectors). The BGP daemon then advertises pod CIDRs to peers, enabling native routing.

---

<a id="how-does-cilium-use-geneve-for-network-policy"></a>

#### Q: How does Cilium use Geneve for network policy?

Cilium is a popular CNI plugin that uses Geneve overlay and eBPF for advanced networking. With Cilium and Geneve, when you send a letter, the building's security system (Cilium agent) checks your ID, looks up the security policy, and attaches security stickers to the outer envelope: "From: frontend-workload", "Policy: allow-frontend-to-backend", "Security Clearance: Level 3". When the letter arrives at the destination building, the security guard there reads the stickers, verifies the policy allows this communication, and only then delivers the letter. If the stickers don't match the policy, the letter is rejected!

<div class="mermaid">
sequenceDiagram
    autonumber
    participant Pod1 as Frontend
    participant Node1 as Node1
    participant NET as Network
    participant Node2 as Node2
    participant Pod2 as Backend

    Pod1->>Node1: Packet to backend

    Note over Node1: Cilium eBPF:<br/>Lookup policy<br/>Add identity to Geneve

    Note over Node1,Node2: GENEVE ENCAPSULATION
    Node1->>NET: Geneve packet<br/>VNI: 5000<br/>TLV: identity, policy
    NET->>Node2: Routed

    Note over Node2: Cilium eBPF:<br/>Read TLV<br/>Verify policy ✅<br/>Decapsulate

    Node2->>Pod2: Packet delivered

    Note over Pod1,Pod2: Policy enforced<br/>via Geneve metadata
</div>

---

## Part 3: Service Mesh and Workload Identity

<a id="what-problem-does-service-mesh-solve"></a>

#### Q: What problem does service mesh solve?

As microservices architectures became the norm in the 2010s, developers faced new challenges that infrastructure-layer networking (VLAN, VXLAN, Geneve) couldn't solve. While overlay networks could route packets between hosts using IP addresses, they operated at Layer 2/3 and couldn't help with application-layer concerns.

**The fundamental problem:** In a microservices world, services are ephemeral—they start, stop, scale, and move constantly. IP addresses change. Network boundaries are fluid. Traditional networking assumptions broke down.

**Why this was painful:**

- **Service discovery**: You couldn't hardcode IPs because containers/pods get new IPs every time they restart or scale. Every service needed to implement its own service discovery (Consul, etcd, custom solutions), leading to inconsistency across teams. When "backend" had 10 instances, which one should you call? How do you load balance? Infrastructure networking could route packets, but couldn't answer "where is the backend service?"

- **Security between services**: Every team was implementing their own TLS, authentication, and authorization logic. Some services used certificates, others used API keys, some had no security at all. This created security gaps, inconsistent implementations, and maintenance nightmares. When you had 50 microservices, you had 50 different security implementations to maintain.

- **Observability**: When a user request failed, which service was the problem? Was it the frontend? The API gateway? The auth service? The database? There was no way to trace a request as it flowed through multiple services. Each service logged independently, but correlating logs across services was nearly impossible. You couldn't see the "big picture" of how services communicated.

- **Traffic management**: Every service needed to implement retry logic, timeouts, circuit breakers, and load balancing. When backend was slow, frontend would retry—but how many times? With what backoff? What if backend was completely down—should you fail fast or keep retrying? Each team made different decisions, leading to cascading failures and inconsistent behavior.

- **Zero-trust security**: Traditional network security relied on firewalls and network boundaries: "trust everything inside the network, block everything outside." But in microservices, there is no "inside"—services move, IPs change, and the network boundary is meaningless. An attacker who compromised one service could access all services on the same network. You needed identity-based security: "trust based on who you are, not where you are."

**The breaking point:** Developers were writing the same networking code (retry logic, TLS, metrics, tracing, service discovery) in every microservice. This was expensive, error-prone, and inconsistent. Service mesh emerged around 2016-2018 (with projects like Linkerd and Istio) to solve these problems by moving networking concerns out of application code and into a dedicated infrastructure layer that worked transparently for all services.

---

<a id="what-is-a-service-mesh"></a>

#### Q: What is a service mesh?

A *service mesh* is a dedicated infrastructure layer that handles service-to-service communication, security, observability, and traffic management for microservices. Unlike VLAN/VXLAN/Geneve which route packets at Layer 2/3 (infrastructure layer) using IP addresses, service mesh routes requests at Layer 7 (application layer) using service names and DNS.

**Architecture:** A service mesh consists of a control plane (e.g., Istio's istiod) that distributes configuration to sidecar proxies (e.g., Envoy) running alongside each application container. The sidecars handle mTLS, routing, and observability transparently. For detailed architecture diagrams, see the <a href="https://istio.io/latest/docs/ops/deployment/architecture/" target="_blank" rel="noopener noreferrer">Istio Architecture documentation</a>.

---

<a id="how-does-service-mesh-relate-to-overlay-networks"></a>

#### Q: How does service mesh relate to overlay networks?

> **💡 Key Insight**
>
> Service mesh works *on top of* overlay networks. You still need VXLAN/Geneve to route packets between hosts/containers at the infrastructure layer. Service mesh then adds Layer 7 routing and capabilities transparently to applications.

While VLAN/VXLAN/Geneve are like the postal service that routes envelopes based on addresses (infrastructure layer), service mesh is like a **smart assistant who reads your letter and routes it based on what you wrote** (application layer). You write "Send this to the accounting department" (service name), and the assistant looks up which building and apartment that is, puts your letter in the right envelope (with security and tracking), and sends it. The assistant also adds a return envelope with your identity certificate, so the recipient knows it's really from you. The postal service (overlay network) still delivers the physical envelope, but the assistant (service mesh) handles the "who talks to whom" and "is this allowed" logic.

---

<a id="what-is-workload-identity"></a>

#### Q: What is workload identity?

*Workload Identity* is a way to identify and authenticate workloads (containers, VMs, processes) using cryptographically verifiable certificates rather than IP addresses. Instead of saying "Allow mail from 10.0.0.5" (IP-based), we now say "Allow mail from the Accounting Department" (identity-based). Each workload gets a **certificate** (like a company ID badge) that proves who they are. When you send a letter, you include a copy of your ID badge in the envelope. The recipient checks: "Is this person from Accounting? Yes, they're allowed to send me mail." It's like moving from checking return addresses (which can be faked) to checking photo IDs (which can't be easily forged).

---

<a id="what-is-the-evolution-of-workload-identity-and-what-problem-was-it-solving"></a>

#### Q: What is the evolution of workload identity and what problem was it solving?

Workload identity evolved to solve the fundamental problem that **IP addresses are not a reliable way to identify workloads** in modern, dynamic environments.

**The old way: IP-based security (pre-2010s)**

- Firewall rules: "Allow 10.0.0.5 to access database on port 5432"
- Network segmentation: "Everything in subnet 10.0.1.0/24 is trusted"
- This worked when servers had static IPs and rarely moved

**Why IP-based security broke down:**

- **Containers and VMs are ephemeral**: A container gets a new IP every time it starts. Your firewall rule for 10.0.0.5 is useless when the container restarts and gets 10.0.0.47.
- **Scaling breaks rules**: When you scale from 1 backend instance to 10, you can't maintain firewall rules for each IP. You'd need to update rules constantly.
- **Workloads move**: A pod moves from Node A to Node B? Its IP changes. Your security rules break.
- **IPs can be spoofed**: An attacker who compromises one workload can spoof IPs to appear as another workload.
- **No context**: An IP address tells you nothing about *what* the workload is or *who* it belongs to. Is 10.0.0.5 the frontend? The backend? A test service? You can't tell from the IP.

**The evolution:**

1. **Service accounts (early 2010s)**: Platforms like Kubernetes introduced service accounts—a step toward identity, but platform-specific. Kubernetes service accounts only work in Kubernetes. AWS IAM roles only work in AWS. No portability.

2. **Workload identity (2018-present)**: Standards like <a href="https://spiffe.io/" target="_blank" rel="noopener noreferrer">SPIFFE</a> emerged to provide portable, verifiable workload identity. Each workload gets a certificate (SVID - SPIFFE Verifiable Identity Document) that proves:
   - **Who it is**: `spiffe://cluster.local/ns/prod/sa/frontend`
   - **Where it came from**: The certificate is cryptographically signed, so it can't be forged
   - **What it can do**: Policies can be written in terms of identity, not IPs

**The breakthrough:** Instead of "Allow IP 10.0.0.5", you now say "Allow workloads with identity `spiffe://cluster.local/ns/prod/sa/frontend`". The identity stays the same even when the IP changes. It works across platforms (Kubernetes, VMs, bare metal). It's cryptographically verifiable, so it can't be spoofed.

---

<a id="when-did-service-mesh-and-workload-identity-get-integrated"></a>

#### Q: When did service mesh and workload identity get integrated?

Service mesh and workload identity evolved separately at first, then became tightly integrated:

- **2016-2017: Early service meshes** (Linkerd 1.0 in 2016, Istio 0.1 in 2017) initially used platform-specific identities:
  - Kubernetes service accounts in Kubernetes environments
  - No standard identity format
  - Identity was tied to the platform

- **2017-2018: SPIFFE emerges**: The SPIFFE project started in 2017 to create a standard for workload identity that works across platforms. SPIFFE provided the foundation, but service meshes weren't using it yet.

- **2019-2020: The integration begins**: Service meshes started adopting SPIFFE:
  - **Istio 1.4 (2020)**: Added SPIFFE integration, allowing Istio to issue SPIFFE SVIDs
  - **Linkerd 2.7+ (2020)**: Integrated SPIFFE for workload identity
  - This was the "marriage" - service meshes could now use standardized, portable workload identity

- **2020-present: Deep integration**: Modern service meshes are built around workload identity:

    - Identity is no longer optional—it's core to how service mesh works
    - mTLS uses workload identity certificates (SVIDs)
    - Authorization policies are written in terms of workload identity
    - Works across Kubernetes, VMs, and bare metal

**Why the integration matters:** Before SPIFFE integration, service meshes were platform-locked. A Kubernetes service account identity couldn't be verified by a VM-based service. With SPIFFE, the same workload identity works everywhere, enabling true multi-platform service mesh deployments and zero-trust security across heterogeneous environments.

---

<a id="what-are-the-identity-models-used-by-service-mesh"></a>

#### Q: What are the identity models used by service mesh?

Service meshes support three main identity models, each with different trade-offs:

- **SPIFFE/SVID**: <a href="https://spiffe.io/" target="_blank" rel="noopener noreferrer">SPIFFE</a> (Secure Production Identity Framework for Everyone) provides a standard, portable workload identity via SVIDs (SPIFFE Verifiable Identity Documents). SPIFFE identities are cryptographically verifiable certificates that work across platforms (Kubernetes, VMs, bare metal). This is the most portable and future-proof approach. Used by Istio (with SPIFFE integration) and Linkerd. **Learn more:** <a href="https://spiffe.io/docs/latest/" target="_blank" rel="noopener noreferrer">SPIFFE Documentation</a>, <a href="https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE.md" target="_blank" rel="noopener noreferrer">SPIFFE Specification</a>

- **Platform service accounts**: Service meshes can use platform-specific service accounts (e.g., Kubernetes service accounts, AWS IAM roles, Azure managed identities) as workload identity. This is simpler to set up but ties you to a specific platform—a Kubernetes service account identity can't be verified by a VM-based service. Good for single-platform deployments.

    **Kubernetes service accounts** are the most common example. Each pod can be assigned a service account, and the service mesh uses this to identify the workload. The service account name (e.g., `frontend-sa`) becomes part of the workload identity. However, this only works within Kubernetes—you can't use a Kubernetes service account identity to authenticate to a VM-based service. **Learn more:** <a href="https://kubernetes.io/docs/concepts/security/service-accounts/" target="_blank" rel="noopener noreferrer">Kubernetes Service Accounts</a>, <a href="https://istio.io/latest/docs/ops/best-practices/security/#configure-service-accounts" target="_blank" rel="noopener noreferrer">Using Service Accounts with Istio</a>, <a href="https://linkerd.io/2.14/features/automatic-mtls/#service-account-identity" target="_blank" rel="noopener noreferrer">Linkerd Service Account Identity</a>

- **Custom identity providers**: Some meshes integrate with cloud provider IAM (AWS IAM, Azure AD, GCP IAM) or custom identity systems. This allows leveraging existing identity infrastructure but requires custom integration work and may not be portable across platforms. **Learn more:** <a href="https://docs.aws.amazon.com/app-mesh/latest/userguide/security_iam.html" target="_blank" rel="noopener noreferrer">AWS App Mesh IAM Integration</a>, <a href="https://learn.microsoft.com/en-us/azure/service-mesh/overview" target="_blank" rel="noopener noreferrer">Azure Service Mesh Identity</a>

---

<a id="what-are-the-constraints-of-service-mesh"></a>

#### Q: What are the constraints of service mesh?

- *CPU overhead*: Every request goes through a proxy, adding 1-5ms latency and consuming CPU
- *Complexity*: Debugging distributed systems with sidecars is harder; requires understanding both application and mesh behavior
- *Latency*: Additional hops add milliseconds (though often acceptable for the benefits)
- *Resource consumption*: Each workload requires a sidecar proxy, increasing memory and CPU usage

---

<a id="do-i-need-service-mesh"></a>

#### Q: Do I need service mesh?

**Short answer: It depends.** Service mesh solves real problems, but it's not always the right solution. Here's a pragmatic guide:

**You probably need service mesh if:**

- You have **10+ microservices** communicating with each other
- You need **zero-trust security** between services (mTLS everywhere)
- You struggle with **observability**—can't trace requests across services, don't know which service is slow
- You're implementing **canary deployments, A/B testing, or traffic splitting** and it's painful
- You're writing the same **retry logic, circuit breakers, or TLS code** in every service
- You have **multi-cloud or multi-cluster** deployments and need unified security/observability
- You're **modernizing legacy apps** and can't change code but need security/observability

**You probably don't need service mesh if:**

- You have **1-3 services** (overkill—use simpler solutions)
- You're running a **monolith or simple 2-tier app** (database + app)
- Your services **don't talk to each other** (just frontend → backend → database)
- You have **simple requirements** and existing tools work fine (API gateway, load balancer, basic monitoring)
- Your team is **small** and can't invest in learning/maintaining service mesh
- **Latency is critical** and you can't afford the 1-5ms overhead
- You're in **early stages** and don't have the problems service mesh solves yet

**The pragmatic approach:**
1. **Start simple**: Use API gateways, load balancers, and basic monitoring first
2. **Add service mesh when you feel the pain**: When you find yourself writing the same networking code in every service, or when observability becomes impossible
3. **Consider alternatives**: API gateways (Kong, Ambassador) can provide some service mesh features without the complexity
4. **Evaluate the trade-offs**: Service mesh adds complexity and overhead. Make sure the benefits (security, observability, traffic management) justify the cost

**Remember**: Service mesh is infrastructure. Like any infrastructure, it should solve problems you actually have, not problems you might have someday. If you're not experiencing the pain points service mesh solves, you probably don't need it yet.

---

<a id="how-does-service-mesh-integrate-with-kubernetes-networking"></a>

#### Q: How does service mesh integrate with Kubernetes networking?

Service mesh works *on top of* Kubernetes networking, adding Layer 7 capabilities. The integration flow:

1. App uses service name (`backend.default.svc.cluster.local` or just `backend`)
2. Sidecar resolves via Kubernetes DNS → Service IP (10.96.0.100)
3. Service discovery → Pod IP (10.244.2.10)
4. Overlay network (VXLAN/Geneve) routes packet to destination pod
5. Service mesh adds mTLS, observability, traffic management

Service mesh leverages Kubernetes service discovery and DNS, then adds identity-based security, request-level observability, and traffic management on top of the existing pod-to-pod networking.

---

<a id="can-you-walk-through-an-example-of-frontend-calling-backend-through-the-mesh-in-kubernetes"></a>

#### Q: Can you walk through an example of frontend calling backend through the mesh in Kubernetes?

This section details the **complete packet journey** in a Kubernetes cluster augmented with a service mesh, illustrating the interplay of all the components discussed so far. The diagram below shows the flow using SPIFFE identities for workload authentication:

<div class="mermaid">
sequenceDiagram
    autonumber
    participant APP1 as Frontend
    participant ENV1 as Envoy1
    participant ENV2 as Envoy2
    participant APP2 as Backend

    Note over APP1: App calls backend

    APP1->>ENV1: GET /api/data

    Note over ENV1: Intercept<br/>DNS lookup<br/>Service discovery<br/>Get identity<br/>Initiate mTLS

    Note over ENV1,ENV2: mTLS HANDSHAKE
    ENV1->>ENV2: ClientHello + SVID
    ENV2->>ENV1: ServerHello + SVID
    Note over ENV1,ENV2: Authenticated

    Note over ENV1,ENV2: ENCRYPTED
    ENV1->>ENV2: Encrypted: GET /api/data

    Note over ENV2: Log metrics<br/>Trace request

    ENV2->>APP2: GET /api/data
    APP2->>ENV2: 200 OK
    ENV2->>ENV1: Encrypted: 200 OK

    ENV1->>APP1: 200 OK

    Note over APP1,APP2: Security & observability<br/>handled by mesh
</div>

---

<a id="whats-the-difference-between-ip-based-and-identity-based-security-in-kubernetes"></a>

#### Q: What's the difference between IP-based and identity-based security in Kubernetes?

The key principle: *identity-based security* replaces IP-based security in modern Kubernetes deployments.

**Old (IP-based):**

- Allow 10.0.0.5
- Deny 10.0.0.6
- Network Policies based on pod IPs (which change constantly)

**New (Identity-based):**

- Allow spiffe://cluster.local/ns/prod/sa/frontend
- Deny spiffe://cluster.local/ns/dev/*
- Authorization policies based on workload identity (which stays constant)

---

<a id="where-do-i-create-identity-based-security-rules-and-who-enforces-them-in-kubernetes"></a>

#### Q: Where do I create identity-based security rules and who enforces them in Kubernetes?

Identity-based security rules are created as **Kubernetes Custom Resources** (CRDs) and enforced by the service mesh data plane (sidecar proxies).

**Where rules are created:**

- **Istio**: Create `AuthorizationPolicy` resources in Kubernetes:
  ```yaml
  apiVersion: security.istio.io/v1beta1
  kind: AuthorizationPolicy
  metadata:
    name: allow-frontend-to-backend
    namespace: prod
  spec:
    selector:
      matchLabels:
        app: backend
    action: ALLOW
    rules:
    - from:
      - source:
          principals: ["cluster.local/ns/prod/sa/frontend"]
  ```

- **Linkerd**: Create `Server` and `ServerAuthorization` resources:
  ```yaml
  apiVersion: policy.linkerd.io/v1beta2
  kind: ServerAuthorization
  metadata:
    name: backend-authz
    namespace: prod
  spec:
    server:
      name: backend
    client:
      meshTLS:
        identities:
        - "frontend.prod.serviceaccount.identity.linkerd.cluster.local"
  ```

- **General pattern**: Rules are defined as YAML manifests and applied via `kubectl apply`, just like any Kubernetes resource.

**Who enforces them:**

- **Service mesh control plane** (istiod, Linkerd control plane): Distributes the rules to all sidecar proxies
- **Sidecar proxies** (Envoy, Linkerd proxy): Enforce the rules at runtime—they intercept traffic, verify workload identity, and allow/deny requests based on the policies
- **No application code changes**: The application doesn't know about the rules—the sidecar proxy handles enforcement transparently

**Example flow:**
1. You create an `AuthorizationPolicy` with `kubectl apply`
2. Istio control plane (istiod) reads the policy and distributes it to all Envoy sidecars
3. When frontend tries to call backend, Envoy sidecar checks: "Does this request come from `spiffe://cluster.local/ns/prod/sa/frontend`?"
4. Envoy looks up the policy: "Yes, frontend is allowed to talk to backend" → Request proceeds
5. If the identity doesn't match, Envoy rejects the request with HTTP 403

**Key insight**: The rules are Kubernetes resources (like Deployments or Services), but enforcement happens in the service mesh data plane (sidecar proxies), not in Kubernetes itself. This gives you identity-based security without modifying application code.

---

<a id="what-is-the-complete-packet-flow-from-app-to-app-in-kubernetes-with-service-mesh"></a>

#### Q: What is the complete packet flow from app to app in Kubernetes with service mesh?

Here's the complete journey of a packet from one pod to another:

<div class="mermaid">
sequenceDiagram
    autonumber
    participant AppA as AppA
    participant SidecarA as SidecarA
    participant Bridge1 as Bridge1
    participant Overlay1 as Overlay1
    participant Network as Network
    participant Overlay2 as Overlay2
    participant Bridge2 as Bridge2
    participant SidecarB as SidecarB
    participant AppB as AppB

    Note over AppA: App sends GET backend

    AppA->>SidecarA: GET backend:8080

    Note over SidecarA: Service discovery<br/>Initiate mTLS

    SidecarA->>Bridge1: Encrypted packet

    Note over Bridge1: Route lookup<br/>Encapsulate overlay

    Bridge1->>Overlay1: Encapsulate<br/>Node1 → Node2

    Overlay1->>Network: IP routing

    Network->>Overlay2: Routed

    Note over Overlay2: Decapsulate

    Overlay2->>Bridge2: Packet

    Bridge2->>SidecarB: Forward

    Note over SidecarB: Verify identity<br/>Decrypt<br/>Check policies

    SidecarB->>AppB: GET /api/data

    AppB->>SidecarB: 200 OK
    SidecarB->>AppA: Encrypted response
</div>

---

## References

### Service Mesh and Identity

- <a href="https://spiffe.io/" target="_blank" rel="noopener noreferrer">SPIFFE</a>: Secure Production Identity Framework for Everyone
- <a href="https://istio.io/" target="_blank" rel="noopener noreferrer">Istio</a>: Connect, Secure, Control, and Observe Services
- <a href="https://linkerd.io/" target="_blank" rel="noopener noreferrer">Linkerd</a>: Ultra Lightweight Service Mesh for Kubernetes

### Container Networking

- <a href="https://github.com/containernetworking/cni/blob/master/SPEC.md" target="_blank" rel="noopener noreferrer">CNI Specification</a>: Container Network Interface
- <a href="https://cilium.io/" target="_blank" rel="noopener noreferrer">Cilium</a>: eBPF-based Networking, Security, and Observability

---

*This article is part of the "Learning in a Hurry" series, designed to help engineers quickly understand complex technical concepts through analogies and practical examples.*


<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
<script>
  mermaid.initialize({ startOnLoad: true });
</script>