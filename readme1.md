# Retina Controllers Documentation

## Overview

The Retina codebase contains **13 controllers** organized into two tiers: **Daemon Controllers** (run on each node) and **Operator Controllers** (centralized, cluster-wide operations).

---

## **DAEMON CONTROLLERS** (Local Node Operations)

### 1. **Namespace Controller**
**Location:** `pkg/controllers/daemon/namespace/namespace_controller.go:36`

**Watches:** Namespace resources with annotation `retina.io/pod-annotation: enabled`

**Reconciles:**
- Maintains cache of annotated namespaces
- Periodically updates metrics module with included namespaces (1s interval)

### 2. **Pod Controller (Daemon)**
**Location:** `pkg/controllers/daemon/pod/controller.go:32`

**Watches:** Pod resources (excluding host network pods)

**Reconciles:**
- Converts Pods → RetinaEndpoint cache objects
- Updates cache with pod IPs and metadata
- Deletes endpoints when pods are removed

### 3. **Service Controller**
**Location:** `pkg/controllers/daemon/service/controller.go:32`

**Watches:** Service resources (excludes headless services with ClusterIP="None")

**Reconciles:**
- Updates cache with Service IPs (IPv4, LoadBalancer IPs)
- Manages service selector metadata
- Handles service deletions

### 4. **Node Controller (Daemon)**
**Location:** `pkg/controllers/daemon/node/controller.go:32`

**Watches:** Node resources

**Reconciles:**
- Updates cache with node IP addresses
- Extracts and stores first node address from status

### 5. **Node Reconciler (Linux)**
**Location:** `pkg/controllers/daemon/nodereconciler/node_controller_linux.go:51`

**Watches:** Node resources (filters on IP/label/annotation changes)

**Reconciles:**
- Registers nodes with datapath handlers
- Updates IP cache (internal + external IPs)
- Assigns node identities (ReservedIdentityHost vs ReservedIdentityRemoteNode)
- Notifies subscribers of node changes

### 6. **RetinaEndpoint Controller (Daemon)**
**Location:** `pkg/controllers/daemon/retinaendpoint/controller.go:32`

**Watches:** RetinaEndpoint CRD (retina.sh API)

**Reconciles:**
- Syncs RetinaEndpoint CRDs to local cache
- Handles endpoint deletions

### 7. **MetricsConfiguration Controller (Daemon)**
**Location:** `pkg/controllers/daemon/metricsconfiguration/metricsconfiguration_controller.go:40`

**Watches:** MetricsConfiguration CRD (operator.retina.sh API)

**Reconciles:**
- Validates metrics configuration specs
- Updates metrics module with config
- Enforces singleton constraint (one config per cluster)
- Updates status (Accepted/Errored)

---

## **OPERATOR CONTROLLERS** (Cluster-Wide Operations)

### 8. **Pod Controller (Operator)**
**Location:** `pkg/controllers/operator/pod/pod_controller.go:31`

**Watches:** Pod resources (only running/deleted pods, filters on IP/label changes)

**Reconciles:**
- Forwards pod events to channel for further processing
- Routes events to RetinaEndpoint creation pipeline

### 9. **Capture Controller**
**Location:** `pkg/controllers/operator/capture/controller.go:61`

**Watches:**
- Capture CRD (retina.sh API)
- Owns: Batch Jobs

**Reconciles:**
- Translates Capture specs → Kubernetes Jobs for packet capture
- Manages capture lifecycle: pending → in-progress → completed/failed
- Handles Azure blob storage integration with managed identities
- Creates/manages secrets for blob credentials
- Updates Capture status based on Job completion
- Cleanup via finalizers

**RBAC:** Full access to Captures, Jobs, Secrets; read access to Nodes/Pods/Namespaces

### 10. **MetricsConfiguration Controller (Operator)**
**Location:** `pkg/controllers/operator/metricsconfiguration/metricsconfiguration_controller.go:36`

**Watches:** MetricsConfiguration CRD (operator.retina.sh API)

**Reconciles:**
- Validates MetricsConfiguration CRDs
- Caches validated configs
- Updates CRD status with validation results
- Enforces singleton pattern

### 11. **RetinaEndpoint Reconciler (Operator)**
**Location:** `pkg/controllers/operator/retinaendpoint/retinaendpoint_controller.go:52`

**Watches:** Pod events from channel (indirect)

**Reconciles:**
- Creates/updates RetinaEndpoint CRDs from Pods
- Populates container info, IPs, labels, owner references
- Deletes RetinaEndpoints when pods deleted
- Retry logic with exponential backoff (max 5 retries)

### 12. **CiliumEndpoint Controller (Linux)**
**Location:** `pkg/controllers/operator/cilium-crds/endpoint/endpoint_controller_linux.go:70`

**Watches:**
- Pod events
- Namespace events
- CiliumEndpoint store events

**Reconciles:**
- Creates/updates CiliumEndpoint CRDs from Pods
- Manages Cilium identity allocation and lifecycle
- Updates IP cache with pod networking info
- Handles namespace label changes → pod reconciliation
- Manages namespace identities for Cilium labels

### 13. **TracesConfiguration Controller**
**Location:** `pkg/controllers/operator/tracesconfiguration/tracesconfiguration_controller.go:36`

**Watches:** TracesConfiguration CRD (operator.retina.sh API)

**Reconciles:** ⚠️ **Currently a stub** - marked with TODO, no implementation yet

---

## **Architecture Pattern**

The system uses a **two-tier architecture**:

### **Daemon Tier**
- Runs on each node as a DaemonSet
- Maintains local node caches for fast lookups
- Handles node-specific resource monitoring
- Watches: Pods, Services, Nodes, Namespaces, RetinaEndpoints, MetricsConfiguration

### **Operator Tier**
- Centralized controllers running as Deployment
- Manages cluster-wide CRDs and coordination
- Handles custom resource lifecycles
- Watches: Captures, RetinaEndpoints, MetricsConfiguration, TracesConfiguration, CiliumEndpoints

### **Common Features**
- All use controller-runtime's `SetupWithManager()` pattern
- Event filtering via predicates to reduce noise
- Caching mechanisms to avoid unnecessary reconciliation
- Status subresources for condition management
- Finalizers for graceful deletion handling

### **Key Integrations**
- **Cilium**: Network policy enforcement and identity management
- **Azure Storage**: Managed storage accounts for packet captures
- **Custom CRDs**: MetricsConfiguration, TracesConfiguration, Capture, RetinaEndpoint
- **Kubernetes Native**: Pods, Services, Nodes, Namespaces, Jobs, Secrets
