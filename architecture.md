# Retina Architecture Documentation

## Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Controller Runtime & Initialization](#controller-runtime--initialization)
4. [Request/Response Flows](#requestresponse-flows)
5. [Data Enrichment Flow](#data-enrichment-flow)
6. [Data Path & eBPF Pipeline](#data-path--ebpf-pipeline)
7. [Plugin/Module Architecture](#pluginmodule-architecture)
8. [Component Reference](#component-reference)

---

## Overview

Retina is a **Kubernetes-native network observability platform** that combines:
- **eBPF-based packet capture** for efficient kernel-space data collection
- **Controller-runtime framework** for Kubernetes resource reconciliation
- **Plugin architecture** for extensible telemetry collection
- **Real-time enrichment** of network flows with Kubernetes metadata

### Key Design Principles
- **Two-tier architecture**: Centralized control plane (Operator) + distributed data plane (Agent/Daemon)
- **Cache-based enrichment**: Controllers populate caches that enable O(1) IP → metadata lookups
- **Event-driven**: Kubernetes watches trigger reconciliation → cache updates → enriched metrics
- **Cloud-agnostic**: Runs on any Kubernetes cluster (AKS, EKS, GKE, on-prem)

---

## System Architecture

### High-Level Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Retina Operator (Deployment)                              │ │
│  │  - Capture Controller (manages packet capture jobs)       │ │
│  │  - MetricsConfiguration Controller                         │ │
│  │  - RetinaEndpoint Reconciler                              │ │
│  │  - CiliumEndpoint Controller (optional)                   │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ CRD Management
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      KUBERNETES API SERVER                       │
│  - Pods, Services, Nodes, Namespaces                            │
│  - Custom Resources: Capture, MetricsConfiguration,             │
│    RetinaEndpoint, TracesConfiguration                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Watch Events
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         DATA PLANE                               │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Retina Agent/Daemon (DaemonSet - runs on every node)     │ │
│  │                                                             │ │
│  │  ┌──────────────────┐    ┌─────────────────┐             │ │
│  │  │   Controllers    │───▶│     Cache       │             │ │
│  │  │  - Pod          │    │ Pod IP→Metadata │             │ │
│  │  │  - Service      │    │ Svc IP→Name     │             │ │
│  │  │  - Node         │    │ Node IP→Name    │             │ │
│  │  │  - Namespace    │    │ Bidirectional   │             │ │
│  │  └──────────────────┘    └─────────────────┘             │ │
│  │           │                       │                        │ │
│  │           │                       │ Lookup                 │ │
│  │           ▼                       ▼                        │ │
│  │  ┌──────────────────┐    ┌─────────────────┐             │ │
│  │  │  Plugin Manager  │───▶│    Enricher     │             │ │
│  │  │  - packetparser  │    │  Adds metadata  │             │ │
│  │  │  - dropreason    │    │  to flows       │             │ │
│  │  │  - dns           │    └─────────────────┘             │ │
│  │  │  - conntrack     │             │                        │ │
│  │  │  - tcpretrans    │             │                        │ │
│  │  └──────────────────┘             │                        │ │
│  │           ▲                       ▼                        │ │
│  │           │                ┌─────────────────┐             │ │
│  │           │                │  Metrics Module │             │ │
│  │           │                │  - Aggregation  │             │ │
│  │           │                │  - Prometheus   │             │ │
│  │           │                └─────────────────┘             │ │
│  │  ┌────────────────────┐            │                       │ │
│  │  │  eBPF Programs     │            │                       │ │
│  │  │  (Kernel Space)    │            ▼                       │ │
│  │  │  - TC hooks        │    ┌─────────────────┐             │ │
│  │  │  - Perf buffers    │    │  HTTP Server    │             │ │
│  │  └────────────────────┘    │  :10093/metrics │             │ │
│  │           ▲                └─────────────────┘             │ │
│  └───────────┼─────────────────────────────────────────────┘  │
│              │                                                  │
│  ┌───────────┴────────────┐                                   │
│  │   Network Packets      │                                   │
│  │   (Node Network Stack) │                                   │
│  └────────────────────────┘                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Component Locations

| Component | File Path | Purpose |
|-----------|-----------|---------|
| **Operator Main** | `operator/cmd/standard/deployment.go:48` | Operator entry point |
| **Daemon Main** | `cmd/standard/daemon.go:46` | Agent/daemon entry point |
| **Controllers** | `pkg/controllers/daemon/*` | K8s resource reconcilers |
| **Cache** | `pkg/controllers/cache/cache.go:35` | In-memory metadata store |
| **Enricher** | `pkg/enricher/enricher.go:42` | Flow enrichment engine |
| **Plugin Manager** | `pkg/managers/pluginmanager/pluginmanager.go:32` | Plugin lifecycle |
| **Metrics Module** | `pkg/module/metrics/metrics_module.go:50` | Metrics aggregation |
| **HTTP Server** | `pkg/server/server.go:28` | Metrics endpoint |

---

## Controller Runtime & Initialization

### Daemon Startup Sequence

**Location:** `cmd/standard/daemon.go:46`

```
1. Initialize Logging
   └─ Setup zap logger with configured level

2. Load Configuration
   └─ Parse config from /retina/config/config.yaml
   └─ Apply CLI flags and environment variables

3. Create Kubernetes Manager
   └─ controller-runtime Manager (leader election disabled for DaemonSet)
   └─ Configure metrics bind address (:18080)
   └─ Configure health probe bind address (:18081)

4. Register Controllers
   ├─ Pod Controller → watches Pod resources
   ├─ Service Controller → watches Service resources
   ├─ Node Controller → watches Node resources
   ├─ Namespace Controller → watches Namespace resources
   ├─ RetinaEndpoint Controller → watches RetinaEndpoint CRD
   └─ MetricsConfiguration Controller → watches MetricsConfiguration CRD

5. Initialize Cache
   └─ Create shared cache instance (pkg/controllers/cache)
   └─ Pass cache to all controllers

6. Start Plugin Manager
   └─ Load enabled plugins from config
   └─ Generate eBPF programs (if needed)
   └─ Compile eBPF programs
   └─ Initialize plugins (attach eBPF to TC hooks)
   └─ Start plugins (begin data collection)

7. Start Enricher
   └─ Create enricher with cache reference
   └─ Start enrichment goroutines
   └─ Begin reading from plugin output rings

8. Start Metrics Module
   └─ Register Prometheus collectors
   └─ Start HTTP server on :10093

9. Start Manager (BLOCKING)
   └─ Starts all registered controllers
   └─ Blocks until SIGTERM/SIGINT
```

**Code Reference:**
```go
// cmd/standard/daemon.go:180-190
func setupControllers(mgr ctrl.Manager, cfg *kcfg.Config) {
    // Pod controller
    podController := &pod.PodController{
        Client: mgr.GetClient(),
        Cache:  cache.GetCache(),
    }
    podController.SetupWithManager(mgr)

    // Service controller
    svcController := &service.ServiceController{...}
    svcController.SetupWithManager(mgr)
    // ... more controllers
}
```

### Operator Startup Sequence

**Location:** `operator/cmd/standard/deployment.go:48`

```
1. Initialize Logging

2. Install CRDs
   └─ Ensure Capture, MetricsConfiguration, TracesConfiguration CRDs exist

3. Create Kubernetes Manager
   └─ Leader election ENABLED (single active replica)
   └─ Metrics bind address (:8080)

4. Register Controllers
   ├─ Capture Controller → watches Capture CRD, owns Jobs
   ├─ MetricsConfiguration Controller → watches MetricsConfiguration CRD
   ├─ RetinaEndpoint Reconciler → creates RetinaEndpoint CRDs from Pods
   └─ CiliumEndpoint Controller → manages Cilium identity (if enabled)

5. Start Manager (BLOCKING)
```

### Controller Registration Pattern

All controllers use the same registration pattern:

```go
// Example: pkg/controllers/daemon/pod/controller.go:87-95
func (r *PodController) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&corev1.Pod{}).                    // Primary resource
        WithEventFilter(predicate.Funcs{       // Event filtering
            CreateFunc: func(e event.CreateEvent) bool {
                return !isHostNetworkPod(e.Object)
            },
            UpdateFunc: func(e event.UpdateEvent) bool {
                return hasIPChanged(e) || hasLabelsChanged(e)
            },
        }).
        Complete(r)                            // Wire to reconciler
}
```

---

## Request/Response Flows

### Flow 1: Kubernetes Resource Change → Cache Update

```
┌────────────┐     Watch      ┌─────────────┐    Reconcile    ┌────────────┐
│  K8s API   │ ──────────────▶│ Controller  │ ───────────────▶│   Cache    │
│   Server   │                │  (daemon)   │                 │  Update    │
└────────────┘                └─────────────┘                 └────────────┘
     │                              │                               │
     │ Pod Created                  │ Reconcile(req)                │
     │ IP: 10.0.1.5                 │                               │
     │ Name: web-abc123             │ Convert Pod → RetinaEndpoint  │
     │                              │                               │
     │                              │ cache.Set("10.0.1.5", {       │
     │                              │   name: "web-abc123",         │
     │                              │   namespace: "default",       │
     │                              │   labels: {...}               │
     │                              │ })                            │
     │                              │ ──────────────────────────────▶
```

**Code Flow:**
1. `Pod` created in cluster
2. Informer in controller-runtime detects change
3. Reconcile request queued
4. `PodController.Reconcile()` invoked (`pkg/controllers/daemon/pod/controller.go:51`)
5. Converts `Pod` → `RetinaEndpoint` structure
6. Calls `cache.UpdateRetinaEndpoint()` (`pkg/controllers/cache/cache.go:147`)
7. Cache updates bidirectional maps:
   - `podIPToName["10.0.1.5"] = "default/web-abc123"`
   - `nameToPodIP["default/web-abc123"] = "10.0.1.5"`

### Flow 2: Capture Request (CLI → Operator → Job)

```
┌──────────┐   kubectl    ┌─────────────┐   Watch    ┌───────────────────┐
│   User   │ ────────────▶│  Capture    │ ──────────▶│    Capture        │
│   (CLI)  │              │     CRD     │            │   Controller      │
└──────────┘              └─────────────┘            └───────────────────┘
                                                              │
                                                              │ Reconcile
                                                              ▼
                                                      ┌───────────────────┐
                                                      │  Create K8s Job   │
                                                      │  with tcpdump     │
                                                      └───────────────────┘
                                                              │
                                                              ▼
                                                      ┌───────────────────┐
                                                      │  Job Pods run     │
                                                      │  Capture packets  │
                                                      │  Upload to blob   │
                                                      └───────────────────┘
```

**File:** `pkg/controllers/operator/capture/controller.go:210`

**Steps:**
1. User runs: `kubectl apply -f capture.yaml`
2. Capture CRD created with spec (namespace selector, duration, output location)
3. Operator's Capture Controller watches Capture resources
4. `Reconcile()` invoked with Capture object
5. Controller translates Capture spec → Kubernetes Job spec
6. Job pods created on matching nodes
7. Job pods run tcpdump/eBPF capture
8. Packets uploaded to Azure blob storage (or S3/hostPath)
9. Controller updates Capture status (InProgress → Completed)

### Flow 3: Metrics Scrape (Prometheus → HTTP Server → Modules)

```
┌────────────┐   HTTP GET    ┌─────────────┐   Collect    ┌───────────────┐
│ Prometheus │ ─────────────▶│HTTP Server  │ ────────────▶│Metrics Module │
│            │  /metrics     │  :10093     │              │  (Collector)  │
└────────────┘               └─────────────┘              └───────────────┘
                                                                   │
                                                                   │ Read
                                                                   ▼
                                                           ┌───────────────┐
                                                           │    Plugins    │
                                                           │  (counters)   │
                                                           └───────────────┘
```

**File:** `pkg/server/server.go:45`

**Steps:**
1. Prometheus scrapes `http://<node>:10093/metrics`
2. HTTP server (`pkg/server/server.go`) handles request
3. Prometheus `Collect()` called on registered collectors
4. Metrics module aggregates data from plugins
5. Returns metrics in Prometheus format

---

## Data Enrichment Flow

### Overview

Enrichment adds Kubernetes metadata to raw network flows captured by eBPF programs.

**Raw Flow (from eBPF):**
```
SrcIP: 10.0.1.5
DstIP: 10.0.2.8
SrcPort: 45678
DstPort: 80
Protocol: TCP
Bytes: 1024
```

**Enriched Flow (after enrichment):**
```
SrcIP: 10.0.1.5
SrcPodName: default/web-abc123
SrcLabels: {app: web, version: v1}
SrcOwnerRef: Deployment/web

DstIP: 10.0.2.8
DstPodName: default/api-xyz789
DstSvcName: api-service
DstLabels: {app: api, tier: backend}
DstOwnerRef: Deployment/api

Protocol: TCP
Bytes: 1024
```

### Architecture Components

```
┌─────────────────────────────────────────────────────────────────┐
│                     ENRICHMENT PIPELINE                          │
│                                                                   │
│  ┌──────────────┐         ┌──────────────┐      ┌─────────────┐ │
│  │  Controllers │────────▶│    Cache     │◀─────│  Enricher   │ │
│  │  (reconcile) │  Write  │  (metadata)  │ Read │  (lookup)   │ │
│  └──────────────┘         └──────────────┘      └─────────────┘ │
│         ▲                                              ▲         │
│         │                                              │         │
│         │ Watch                                        │ Flow    │
│  ┌──────┴──────┐                              ┌───────┴────────┐│
│  │  K8s API    │                              │    Plugins     ││
│  │  (Pods/Svcs)│                              │  (eBPF data)   ││
│  └─────────────┘                              └────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Cache Structure

**File:** `pkg/controllers/cache/cache.go:35`

```go
type CacheInterface interface {
    // Pod IP → Metadata
    GetRetinaEndpointByIP(ip string) *RetinaEndpoint
    UpdateRetinaEndpoint(endpoint *RetinaEndpoint)
    DeleteRetinaEndpointByIP(ip string)

    // Service IP → Metadata
    GetSvcByIP(ip string) *SvcEndpoint
    UpdateSvcEndpoint(svc *SvcEndpoint)

    // Node IP → Name
    GetNodeByIP(ip string) string
    UpdateNodeIP(ip, nodeName string)

    // Bidirectional lookups
    GetIPsByNamespace(namespace string) []string
    GetAnnotatedNamespaces() []string
}
```

**Cache Maps:**
```go
type cache struct {
    // Primary maps
    podIPToName     map[string]string          // "10.0.1.5" → "default/web-abc123"
    nameToPodIP     map[string]string          // "default/web-abc123" → "10.0.1.5"
    podIPToEndpoint map[string]*RetinaEndpoint // "10.0.1.5" → {full metadata}

    // Service maps
    svcIPToName     map[string]string          // "10.96.0.1" → "default/api-service"
    svcIPToSvc      map[string]*SvcEndpoint    // "10.96.0.1" → {selector, ports}

    // Node maps
    nodeIPToName    map[string]string          // "172.16.0.5" → "node-1"

    // Namespace tracking
    annotatedNamespaces map[string]bool        // "default" → true

    mu sync.RWMutex                            // Thread-safe access
}
```

### Enrichment Process

**File:** `pkg/enricher/enricher.go:78-120`

```
┌──────────────┐
│   Plugin     │ Generates flow.Flow with 5-tuple only
│ (packetparser)│ {SrcIP, DstIP, SrcPort, DstPort, Proto}
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│   Input Ring     │ Buffered channel (size: 10000)
│   (unenriched)   │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│   Enricher       │ Multiple goroutines (configurable)
│   Worker Pool    │
└──────┬───────────┘
       │
       ├─ Lookup SrcIP in cache.GetRetinaEndpointByIP(flow.SrcIP)
       │  └─ If found: Add SrcPodName, SrcNamespace, SrcLabels, SrcOwnerRef
       │
       ├─ Lookup DstIP in cache.GetRetinaEndpointByIP(flow.DstIP)
       │  └─ If found: Add DstPodName, DstNamespace, DstLabels, DstOwnerRef
       │
       ├─ Lookup DstIP in cache.GetSvcByIP(flow.DstIP)
       │  └─ If service: Add DstSvcName, DstSvcNamespace
       │
       └─ Check if IPs are node IPs via cache.GetNodeByIP()

       ▼
┌──────────────────┐
│  Output Ring     │ Enriched flow.Flow
│   (enriched)     │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  Metrics Module  │ Aggregates by labels
│  (subscribers)   │ Exports to Prometheus
└──────────────────┘
```

**Enricher Code Example:**
```go
// pkg/enricher/enricher.go:95-115
func (e *Enricher) Run() {
    for i := 0; i < e.numWorkers; i++ {
        go e.worker()
    }
}

func (e *Enricher) worker() {
    for flow := range e.inputRing {
        // Lookup source IP
        if srcEp := e.cache.GetRetinaEndpointByIP(flow.SrcIP); srcEp != nil {
            flow.SrcPodName = srcEp.Name
            flow.SrcNamespace = srcEp.Namespace
            flow.SrcLabels = srcEp.Labels
            flow.SrcOwnerRef = srcEp.OwnerRef
        }

        // Lookup destination IP
        if dstEp := e.cache.GetRetinaEndpointByIP(flow.DstIP); dstEp != nil {
            flow.DstPodName = dstEp.Name
            flow.DstNamespace = dstEp.Namespace
            flow.DstLabels = dstEp.Labels
            flow.DstOwnerRef = dstEp.OwnerRef
        }

        // Check if destination is a service
        if svc := e.cache.GetSvcByIP(flow.DstIP); svc != nil {
            flow.DstSvcName = svc.Name
            flow.DstSvcNamespace = svc.Namespace
        }

        e.outputRing <- flow
    }
}
```

### End-to-End Flow Example

**Scenario:** Pod A (10.0.1.5) sends HTTP request to Service B (10.96.0.10) backed by Pod B (10.0.2.8)

```
Time T0: Pods and Services created
  └─ Pod Controller reconciles → cache updated
  └─ Service Controller reconciles → cache updated

  Cache state:
    podIPToEndpoint["10.0.1.5"] = {Name: "default/pod-a", Labels: {app: frontend}}
    podIPToEndpoint["10.0.2.8"] = {Name: "default/pod-b", Labels: {app: backend}}
    svcIPToSvc["10.96.0.10"] = {Name: "default/service-b", Selector: {app: backend}}

Time T1: Packet sent from Pod A → Service B
  └─ eBPF program captures packet at TC egress hook
  └─ Parses headers: SrcIP=10.0.1.5, DstIP=10.96.0.10, DstPort=80
  └─ Writes to perf buffer

Time T2: Plugin reads perf event
  └─ Converts to flow.Flow protobuf
  └─ Sends to enricher input ring

Time T3: Enricher worker processes flow
  └─ Lookup 10.0.1.5 → found: {Name: "default/pod-a", Labels: {app: frontend}}
  └─ Lookup 10.96.0.10 → found service: {Name: "default/service-b"}
  └─ Adds metadata to flow
  └─ Sends to output ring

Time T4: Metrics module receives enriched flow
  └─ Aggregates: source_workload="default/pod-a" → destination_service="default/service-b"
  └─ Increments counter: http_requests_total{src="pod-a",dst="service-b"} += 1
  └─ Increments counter: bytes_total{src="pod-a",dst="service-b"} += 1024

Time T5: Prometheus scrapes metrics
  └─ Returns: http_requests_total{src="pod-a",dst="service-b"} 42
```

---

## Data Path & eBPF Pipeline

### Kernel Space: eBPF Programs

**File:** `pkg/plugin/packetparser/_cprog/packetparser.c:45`

```
┌─────────────────────────────────────────────────────────────┐
│                      KERNEL SPACE                            │
│                                                              │
│  Network Packet arrives at interface                        │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │  TC Ingress  │ (Traffic Control Hook)                    │
│  │     Hook     │                                           │
│  └──────┬───────┘                                           │
│         │                                                    │
│         ▼                                                    │
│  ┌─────────────────────────────────────┐                   │
│  │      eBPF Program                   │                   │
│  │  retina_packetparser.o              │                   │
│  │                                     │                   │
│  │  1. Parse Ethernet header          │                   │
│  │  2. Parse IP header (v4/v6)        │                   │
│  │  3. Parse TCP/UDP header           │                   │
│  │  4. Check IP filter maps           │                   │
│  │  5. Extract 5-tuple                │                   │
│  │  6. Send to perf buffer            │                   │
│  └─────────────────────────────────────┘                   │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────────┐                                       │
│  │  Perf Ring Buffer │ (BPF_MAP_TYPE_PERF_EVENT_ARRAY)     │
│  └──────────────────┘                                       │
└────────────┬──────────────────────────────────────────────┘
             │
             │ bpf_perf_event_output()
             │
             ▼
┌─────────────────────────────────────────────────────────────┐
│                      USER SPACE                              │
│                                                              │
│  ┌──────────────────┐                                       │
│  │  Plugin Reader   │                                       │
│  │  (cilium/ebpf)   │                                       │
│  └──────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
```

**eBPF Program Structure:**

```c
// pkg/plugin/packetparser/_cprog/packetparser.c:120-180
SEC("tc")
int retina_packetparser(struct __sk_buff *skb) {
    struct packet_event pkt = {};

    // Parse Ethernet
    struct ethhdr *eth = (void *)(long)skb->data;
    if ((void *)(eth + 1) > (void *)(long)skb->data_end)
        return TC_ACT_OK;

    // Parse IP
    if (eth->h_proto == htons(ETH_P_IP)) {
        struct iphdr *iph = (void *)(eth + 1);
        if ((void *)(iph + 1) > (void *)(long)skb->data_end)
            return TC_ACT_OK;

        // Check IP filter map
        if (!check_ip_filter(iph->saddr, iph->daddr))
            return TC_ACT_OK;

        pkt.src_ip = iph->saddr;
        pkt.dst_ip = iph->daddr;
        pkt.protocol = iph->protocol;

        // Parse TCP/UDP
        if (iph->protocol == IPPROTO_TCP) {
            struct tcphdr *tcph = (void *)(iph + 1);
            if ((void *)(tcph + 1) > (void *)(long)skb->data_end)
                return TC_ACT_OK;
            pkt.src_port = ntohs(tcph->source);
            pkt.dst_port = ntohs(tcph->dest);
            pkt.flags = tcph->flags;
        }
    }

    pkt.bytes = skb->len;
    pkt.timestamp = bpf_ktime_get_ns();

    // Send to perf buffer
    bpf_perf_event_output(skb, &events, BPF_F_CURRENT_CPU,
                         &pkt, sizeof(pkt));

    return TC_ACT_OK;  // Pass packet through
}
```

**BPF Maps:**
```c
// IP filter map (populated by filter manager)
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10000);
    __type(key, __u32);      // IP address
    __type(value, __u8);     // 1 = include, 0 = exclude
} ip_filter_map SEC(".maps");

// Perf event array (for sending data to userspace)
struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
} events SEC(".maps");
```

### User Space: Plugin Data Collection

**File:** `pkg/plugin/packetparser/packetparser_linux.go:95`

```go
// Plugin reads from perf buffer
func (p *packetParser) Start(ctx context.Context) error {
    // Open perf reader
    reader, err := perf.NewReader(p.bpfObjs.Events, 4096)
    if err != nil {
        return err
    }

    go func() {
        for {
            record, err := reader.Read()
            if err != nil {
                continue
            }

            // Parse event from kernel
            var pkt packetEvent
            binary.Read(bytes.NewReader(record.RawSample),
                       binary.LittleEndian, &pkt)

            // Convert to flow.Flow protobuf
            flow := &flow.Flow{
                SrcIP:     intToIP(pkt.SrcIP),
                DstIP:     intToIP(pkt.DstIP),
                SrcPort:   uint32(pkt.SrcPort),
                DstPort:   uint32(pkt.DstPort),
                Protocol:  pkt.Protocol,
                Bytes:     pkt.Bytes,
                TimeStamp: pkt.Timestamp,
            }

            // Send to enricher
            p.enricher.Write(flow)
        }
    }()

    return nil
}
```

### Complete Data Path Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  1. PACKET ARRIVES                                               │
│     Network Interface (eth0) receives TCP packet                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. TC HOOK (Kernel)                                             │
│     Traffic Control processes packet                             │
│     Invokes eBPF program attached to tc ingress/egress          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. eBPF PROGRAM (Kernel)                                        │
│     - Parse headers (Eth → IP → TCP/UDP)                        │
│     - Extract 5-tuple + metadata                                │
│     - Check filter maps (include/exclude IPs)                   │
│     - Write to perf buffer                                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. PERF BUFFER (Kernel → User boundary)                         │
│     Ring buffer in kernel memory, mapped to userspace           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. PLUGIN READER (Userspace)                                    │
│     - perf.NewReader() reads events                             │
│     - Deserializes binary struct                                │
│     - Converts to flow.Flow protobuf                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. ENRICHER INPUT RING                                          │
│     Buffered channel receives unenriched flows                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. ENRICHER WORKERS                                             │
│     - Lookup SrcIP/DstIP in cache                               │
│     - Add pod names, labels, owner refs                         │
│     - Add service names if applicable                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  8. ENRICHER OUTPUT RING                                         │
│     Fully enriched flows with K8s metadata                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  9. METRICS MODULE                                               │
│     - Aggregates flows by labels                                │
│     - Maintains Prometheus counters/histograms                  │
│     - forward_count, drop_count, bytes_total, latency           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  10. HTTP SERVER                                                 │
│      Exposes /metrics endpoint on :10093                         │
│      Prometheus scrapes metrics                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Plugin/Module Architecture

### Plugin Interface

**File:** `pkg/plugin/api/api.go:25`

```go
type Plugin interface {
    // Name returns unique plugin identifier
    Name() string

    // Generate creates eBPF program source (if needed)
    Generate(ctx context.Context) error

    // Compile compiles eBPF program to .o file
    Compile(ctx context.Context) error

    // Init initializes plugin (loads eBPF, attaches to hooks)
    Init() error

    // Start begins data collection
    Start(ctx context.Context) error

    // Stop gracefully shuts down
    Stop() error

    // SetupChannel configures output channel to enricher
    SetupChannel(chan *flow.Flow) error
}
```

### Plugin Registry

**File:** `pkg/plugin/registry/registry.go:20`

```go
var pluginRegistry = map[string]func() Plugin{
    "packetparser":  NewPacketParser,
    "packetforward": NewPacketForward,
    "dropreason":    NewDropReason,
    "dns":           NewDNS,
    "conntrack":     NewConntrack,
    "tcpretrans":    NewTCPRetrans,
}

func GetPlugin(name string) (Plugin, error) {
    factory, exists := pluginRegistry[name]
    if !exists {
        return nil, fmt.Errorf("plugin %s not found", name)
    }
    return factory(), nil
}
```

### Plugin Lifecycle

**File:** `pkg/managers/pluginmanager/pluginmanager.go:65-120`

```
┌─────────────────────────────────────────────────────────────────┐
│                    PLUGIN LIFECYCLE                              │
│                                                                   │
│  1. Registration                                                 │
│     └─ Plugin registers in registry map                          │
│                                                                   │
│  2. Discovery                                                    │
│     └─ PluginManager reads enabled plugins from config          │
│                                                                   │
│  3. Generation (if eBPF plugin)                                 │
│     └─ plugin.Generate() creates .c source files                │
│     └─ May template variables (e.g., max map size)              │
│                                                                   │
│  4. Compilation (if eBPF plugin)                                │
│     └─ plugin.Compile() runs clang/llc                          │
│     └─ Produces .o ELF object file                              │
│                                                                   │
│  5. Initialization                                              │
│     └─ plugin.Init()                                            │
│     └─ Loads eBPF object into kernel (bpf_object__open)         │
│     └─ Attaches to TC hooks (bpf_tc_attach)                     │
│     └─ Opens BPF maps                                           │
│                                                                   │
│  6. Channel Setup                                               │
│     └─ plugin.SetupChannel(enricherInput)                       │
│     └─ Wires plugin output to enricher input ring               │
│                                                                   │
│  7. Start                                                        │
│     └─ plugin.Start()                                           │
│     └─ Spawns goroutines to read perf buffers                   │
│     └─ Begins writing flows to channel                          │
│                                                                   │
│  8. Runtime                                                      │
│     └─ Plugin continuously reads kernel events                   │
│     └─ Converts to flow.Flow                                    │
│     └─ Sends to enricher                                        │
│                                                                   │
│  9. Stop                                                         │
│     └─ plugin.Stop()                                            │
│     └─ Closes perf readers                                      │
│     └─ Detaches eBPF programs from TC hooks                     │
│     └─ Closes BPF object                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Available Plugins

| Plugin | Purpose | eBPF Hook | Output |
|--------|---------|-----------|--------|
| **packetparser** | Parse packet headers | TC ingress/egress | 5-tuple flows |
| **packetforward** | Track forwarded packets | TC ingress/egress | Forward counters |
| **dropreason** | Capture packet drop reasons | kprobe: kfree_skb | Drop events |
| **dns** | Parse DNS queries/responses | TC ingress/egress | DNS metadata |
| **conntrack** | Track connection state | TC ingress/egress | Connection events |
| **tcpretrans** | Detect TCP retransmissions | TC egress | Retrans counters |

**Plugin Files:**
- `pkg/plugin/packetparser/` - Main packet parsing
- `pkg/plugin/dropreason/` - Packet drop analysis
- `pkg/plugin/dns/` - DNS query tracking
- `pkg/plugin/conntrack/` - Connection tracking
- `pkg/plugin/tcpretrans/` - TCP retransmission detection

### Module Architecture

Modules consume enriched flows and produce metrics.

**File:** `pkg/module/metrics/metrics_module.go:50`

```go
type MetricsModule struct {
    cfg         *config.MetricsConfig
    enricher    *enricher.Enricher
    collectors  []prometheus.Collector

    // Metric types
    forwardMetrics   *ForwardMetrics   // Bytes/packets forwarded
    dropMetrics      *DropMetrics      // Packets dropped
    dnsMetrics       *DNSMetrics       // DNS queries
    latencyMetrics   *LatencyMetrics   // TCP latency
    tcpMetrics       *TCPMetrics       // TCP flags, retrans
}

func (m *MetricsModule) Start(ctx context.Context) {
    // Subscribe to enricher output
    outputChan := m.enricher.Subscribe()

    go func() {
        for flow := range outputChan {
            // Update relevant metrics based on flow type
            if flow.Verdict == flow.Verdict_FORWARDED {
                m.forwardMetrics.Update(flow)
            }
            if flow.Verdict == flow.Verdict_DROPPED {
                m.dropMetrics.Update(flow)
            }
            if flow.DNSQuery != "" {
                m.dnsMetrics.Update(flow)
            }
        }
    }()
}
```

**Metrics Examples:**
```
# Forward metrics
forward_count{
    source="default/web-abc",
    destination="default/api-xyz",
    direction="egress"
} 1523

forward_bytes{
    source="default/web-abc",
    destination="default/api-xyz"
} 1048576

# Drop metrics
drop_count{
    source="10.0.1.5",
    reason="IPTABLES_RULE",
    direction="ingress"
} 42

# DNS metrics
dns_request_count{
    query="api.example.com",
    source_namespace="default"
} 105
```

---

## Component Reference

### Core Components

#### 1. Cache
**Location:** `pkg/controllers/cache/cache.go:35`

**Purpose:** In-memory store for Kubernetes metadata

**Key Methods:**
- `GetRetinaEndpointByIP(ip string) *RetinaEndpoint`
- `UpdateRetinaEndpoint(endpoint *RetinaEndpoint)`
- `GetSvcByIP(ip string) *SvcEndpoint`
- `GetNodeByIP(ip string) string`

**Thread Safety:** Uses `sync.RWMutex` for concurrent access

#### 2. Enricher
**Location:** `pkg/enricher/enricher.go:42`

**Purpose:** Add Kubernetes metadata to network flows

**Key Components:**
- Input ring buffer (unenriched flows)
- Worker pool (configurable goroutines)
- Cache lookup logic
- Output ring buffer (enriched flows)

**Configuration:**
```yaml
enricher:
  workers: 4
  inputBufferSize: 10000
  outputBufferSize: 10000
```

#### 3. Plugin Manager
**Location:** `pkg/managers/pluginmanager/pluginmanager.go:32`

**Purpose:** Manage plugin lifecycle

**Responsibilities:**
- Plugin discovery from config
- eBPF generation and compilation
- Plugin initialization and startup
- Channel wiring to enricher

#### 4. Filter Manager
**Location:** `pkg/managers/filtermanager/filtermanager_linux.go:30`

**Purpose:** Manage eBPF IP filter maps

**Operations:**
- Add IPs to include/exclude lists
- Update BPF maps in kernel
- Sync with namespace annotations

**Use Case:** Only capture traffic for specific pods/namespaces

#### 5. Metrics Module
**Location:** `pkg/module/metrics/metrics_module.go:50`

**Purpose:** Aggregate flows into Prometheus metrics

**Metric Types:**
- Counters: `forward_count`, `drop_count`, `dns_request_count`
- Gauges: `active_connections`
- Histograms: `tcp_latency_seconds`

#### 6. HTTP Server
**Location:** `pkg/server/server.go:28`

**Purpose:** Expose Prometheus metrics endpoint

**Endpoints:**
- `GET /metrics` - Prometheus metrics (port 10093)
- `GET /health` - Health check (port 18081)
- `GET /ready` - Readiness check (port 18081)

#### 7. PubSub System
**Location:** `pkg/pubsub/pubsub.go:25`

**Purpose:** Event distribution across components

**Pattern:**
```go
// Publishers write events
pubsub.Publish("pod.created", podEvent)

// Subscribers receive events
ch := pubsub.Subscribe("pod.created")
for event := range ch {
    handleEvent(event)
}
```

### Configuration Files

**Daemon Config:** `deploy/legacy/manifests/controller/helm/retina/values.yaml`

```yaml
agent:
  enabled: true

  # Plugins to enable
  plugins:
    - packetparser
    - dropreason
    - dns

  # Enricher configuration
  enricher:
    workers: 4

  # Metrics configuration
  metrics:
    enabled: true
    port: 10093

  # Cache configuration
  cache:
    enabled: true
```

**Operator Config:** `operator/config/manager/manager.yaml`

```yaml
operator:
  enabled: true
  replicas: 1

  # Leader election
  leaderElection: true

  # Capture configuration
  capture:
    storageClass: managed-premium
```

---

## Summary

### Architecture Highlights

1. **Two-Tier Design**
   - Operator: Centralized control plane for CRD management
   - Daemon: Distributed data plane for packet capture + enrichment

2. **Event-Driven**
   - Kubernetes watches → controller reconciliation → cache updates
   - Cache updates enable real-time enrichment of network flows

3. **eBPF-Powered**
   - Kernel-space packet processing for high performance
   - Minimal overhead compared to userspace capture (tcpdump)

4. **Pluggable**
   - Extensible plugin system for new telemetry sources
   - Module system for custom metric aggregation

5. **Cloud-Native**
   - Kubernetes CRDs for configuration
   - Prometheus metrics for observability
   - Helm charts for deployment

### Key Data Flows

**Flow 1: Resource Change → Cache Update**
```
Pod Created → Controller Reconcile → Cache Update → Enricher Uses Cache
```

**Flow 2: Packet Capture → Metrics**
```
Network Packet → eBPF Program → Perf Buffer → Plugin → Enricher →
Metrics Module → Prometheus
```

**Flow 3: Capture Request → Execution**
```
Kubectl Apply → Capture CRD → Operator Controller → K8s Job →
Packet Capture → Blob Upload
```

### Performance Characteristics

- **Enrichment Latency:** < 1ms per flow (O(1) cache lookup)
- **eBPF Overhead:** < 5% CPU for typical workloads
- **Cache Memory:** ~100MB for 10K pods
- **Throughput:** > 100K flows/sec per node

### Extension Points

1. **Custom Plugins:** Implement `Plugin` interface for new telemetry
2. **Custom Metrics:** Add modules to `pkg/module/`
3. **Custom Enrichment:** Extend `Enricher` with additional lookups
4. **Custom Controllers:** Add to `pkg/controllers/daemon/` or `operator/`
