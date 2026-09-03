---
title: gateway-api-gateway-customization
authors:
  - "@gcs278"
reviewers:
  - "@Miciah"
  - "@candita"
  - "@rikatz"
approvers:
  - "@Miciah"
api-approvers:
  - TBD
creation-date: 2026-09-01
last-updated: 2026-09-03
status: provisional
tracking-link:
  - https://redhat.atlassian.net/browse/NE-2698
see-also:
  - "/enhancements/ingress/gateway-api-with-cluster-ingress-operator.md"
  - "/enhancements/ingress/gateway-api-crd-management-mode.md"
replaces: []
superseded-by: []
---

# Gateway API Gateway Customization

## Summary

This enhancement introduces `GatewayParameters`, a new cluster-scoped
CRD in the `operator.openshift.io` API group, and establishes the first
OpenShift-native configuration layer for Gateway API infrastructure
customization. A `GatewayClass` references a `GatewayParameters`
instance via `spec.parametersRef`, and the Cluster Ingress Operator
(CIO) reconciles it into the implementation-specific configuration —
today, the OSSM GatewayClass defaults ConfigMap. The API is
implementation-agnostic: the OpenShift types abstract the underlying
mechanism so that future changes to the Gateway API implementation do
not require API changes or user migrations.

This EP establishes the plumbing and implements two concrete use cases:
ClusterIP service type and `externalTrafficPolicy: Local`. The
`GatewayParameters` CRD is designed to be extended with additional
fields (resource requests, node placement, proxy configuration) in
follow-on EPs without breaking changes to the GatewayClass or Gateway
resources.

## Motivation

OpenShift ships Gateway API via OSSM, which supports customization of
the backing Service and proxy Deployment through implementation-specific
mechanisms (GatewayClass defaults ConfigMap, alpha annotations). These
mechanisms are undocumented, unsupported for end users, and tied to
Istio internals. There is no OpenShift API that:

- Provides a stable, validated interface for Gateway infrastructure
  customization
- Abstracts implementation details so customizations survive Gateway
  API implementation changes
- Integrates with CIO to derive platform-specific configuration
  (cloud LB annotations, OVN settings) automatically

As a result, when a user creates a Gateway using the `openshift-default`
GatewayClass, the implementation always provisions an external
LoadBalancer service, and there is no supported path to change this.
Specific gaps that generate support exceptions today:

- ClusterIP service type for bare-metal deployments fronted by an OCP
  Route, where no hardware or software load balancer is available
- `externalTrafficPolicy: Local` for zone-aware or BGP-based topologies
  where cross-zone hops must be avoided and source IP must be preserved

Users who need these configurations must use the Istio ClusterIP alpha
annotation or manually patch the Service — both fragile approaches that
are overwritten on reconciliation and unsupported for production use.

### User Stories

#### Story 1: ClusterIP Gateway on Bare Metal

As a cluster administrator running on bare metal without a hardware
load balancer, I want to create a GatewayClass that provisions Gateways
with a ClusterIP service so that I can front the Gateway with an OCP
Route using the existing HAProxy ingress infrastructure.

#### Story 2: Zone-Aware External Gateway with ETP Local

As a cluster administrator running a cluster across multiple
availability zones with BGP-based networking, I want to configure
`externalTrafficPolicy: Local` on my GatewayClass so that traffic
arriving in a given zone is served by the proxy pod in that same zone,
avoiding cross-zone hops and preserving source IP.

#### Story 3: Internal LoadBalancer Gateway

As a cluster administrator, I want to create a GatewayClass that
provisions Gateways with an internal LoadBalancer service, including
the correct platform-specific internal annotations, so that I can
expose services only within my cloud provider's private network.

### Goals

- Introduce a `GatewayParameters` CRD that a `GatewayClass` references
  via `spec.parametersRef` to configure its service topology.
- CIO reconciles `GatewayParameters` → OSSM GatewayClass defaults
  ConfigMap (labeled `gateway.istio.io/defaults-for-class`), translating
  the OpenShift API into the implementation-specific ConfigMap format.
- Implement `endpointPublishingStrategy` for service type
  (LoadBalancer/NodePort/ClusterIP) and `externalTrafficPolicy`
  (Local/Cluster).
- CIO derives platform-specific service annotations automatically from
  cluster infrastructure, as it does for IngressControllers.
- CIO manages DNS for GatewayClasses with LoadBalancer service type;
  not for ClusterIP or NodePort.
- The existing `openshift-default` GatewayClass is unchanged.
- Design the CRD for future extension (resources, nodePlacement) without
  breaking API changes.

### Non-Goals

- Customizing the `openshift-default` GatewayClass.
- Exposing the OSSM GatewayClass defaults ConfigMap (`gateway.istio.io/defaults-for-class`)
  as a supported API for end users. It is an implementation detail owned
  and managed exclusively by CIO.
- Using or supporting the Istio ClusterIP alpha annotation
  (`networking.istio.io/service-type`) as a supported mechanism for
  service type customization. This annotation is an unsupported private
  API and is superseded by this enhancement.
- Per-Gateway resource overrides. `GatewayParameters` configures
  class-level defaults shared by all Gateways referencing the class.
- Gateway API implementation configuration (logging, control plane
  settings).
- `trafficDistribution` on the Service. When `externalTrafficPolicy: Local`
  is set it is superseded; for ClusterIP traffic to the Gateway it is
  a no-op for the primary use cases.
- HPA configuration (deferred to a follow-on).
- Node placement and tolerations (deferred to a follow-on).

## Proposal

A new cluster-scoped CRD `GatewayParameters`
(`operator.openshift.io/v1alpha1`) is introduced. A cluster
administrator creates a `GatewayClass` with `spec.parametersRef`
pointing to a `GatewayParameters` instance, and creates the
`GatewayParameters` CR to configure the desired service topology.

CIO watches GatewayClasses whose `spec.controllerName` matches the
OpenShift controller name. For any such GatewayClass with a
`spec.parametersRef` pointing to a `GatewayParameters` CR, CIO:

1. Reads the `GatewayParameters` CR.
2. Derives platform-specific service annotations from the cluster's
   `infrastructure.config.openshift.io/cluster` resource.
3. Creates or updates a ConfigMap in the `openshift-ingress` namespace
   with the label `gateway.istio.io/defaults-for-class: <gatewayclass-name>`.
   OSSM reads this ConfigMap to apply service type, annotations, and ETP
   to all Gateways referencing the class.
4. Manages DNS for `LoadBalancerService` type only.

CIO never mutates the user's `Gateway` or `GatewayClass` resources.

### Workflow Description

#### ClusterIP Gateway (bare-metal / OCP Route topology)

1. The cluster administrator creates a `GatewayParameters` CR:

   ```yaml
   apiVersion: operator.openshift.io/v1alpha1
   kind: GatewayParameters
   metadata:
     name: clusterip-params
   spec:
     endpointPublishingStrategy:
       type: ClusterIPService
   ```

2. The cluster administrator creates a `GatewayClass` referencing it:

   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: GatewayClass
   metadata:
     name: openshift-clusterip
   spec:
     controllerName: openshift.io/gateway-controller/v1
     parametersRef:
       group: operator.openshift.io
       kind: GatewayParameters
       name: clusterip-params
   ```

3. CIO creates a ConfigMap in `openshift-ingress`:

   ```yaml
   metadata:
     labels:
       gateway.istio.io/defaults-for-class: openshift-clusterip
   data:
     service: |
       spec:
         type: ClusterIP
   ```

4. The cluster administrator creates a Gateway referencing
   `openshift-clusterip`. OSSM provisions an Envoy Deployment and a
   ClusterIP Service. No cloud LB is created, no DNS is managed.

5. The cluster administrator creates an OCP Route pointing at the
   ClusterIP Service to expose the Gateway externally via HAProxy.

#### Zone-Aware LoadBalancer with ETP Local

1. The cluster administrator creates a `GatewayParameters` CR:

   ```yaml
   apiVersion: operator.openshift.io/v1alpha1
   kind: GatewayParameters
   metadata:
     name: external-zone-aware
   spec:
     endpointPublishingStrategy:
       type: LoadBalancerService
       loadBalancer:
         scope: External
         endpointTrafficPolicy: Local
   ```

2. The cluster administrator creates a GatewayClass referencing it
   with `spec.parametersRef` (same pattern as above).

3. CIO creates the ConfigMap with `service.type: LoadBalancer`,
   the platform external LB annotation, `externalTrafficPolicy: Local`,
   and the OVN `local-with-fallback` annotation where applicable.

4. Gateways referencing this class get a LoadBalancer Service with ETP
   Local. CIO manages DNS.

### API Extensions

#### `GatewayParameters` CRD

```go
// GatewayParameters configures the infrastructure provisioned by the
// OpenShift ingress operator for a GatewayClass that references it via
// spec.parametersRef.
//
// This CRD is designed to be extended to support Gateway-level
// customization in a future release, pending upstream plumbing support
// for non-mutating per-Gateway configuration injection.
//
// +kubebuilder:object:root=true
// +kubebuilder:resource:scope=Cluster
// +kubebuilder:subresource:status
// +openshift:enable:FeatureGate=GatewayClassParameters
type GatewayParameters struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   GatewayParametersSpec   `json:"spec"`
    Status GatewayParametersStatus `json:"status,omitempty"`
}

type GatewayParametersSpec struct {
    // endpointPublishingStrategy defines how the Gateway's backing
    // Service is provisioned. When omitted, defaults to an external
    // LoadBalancer with platform defaults (matching openshift-default
    // behavior).
    //
    // +optional
    EndpointPublishingStrategy *GatewayEndpointPublishingStrategy `json:"endpointPublishingStrategy,omitempty"`

    // Future fields (not in this EP):
    //   resources     *corev1.ResourceRequirements
    //   nodePlacement *NodePlacement
}
```

#### `GatewayEndpointPublishingStrategy`

```go
// +union
// +kubebuilder:validation:XValidation:rule="self.type != 'LoadBalancerService' || has(self.loadBalancer)",message="loadBalancer is required when type is LoadBalancerService"
// +kubebuilder:validation:XValidation:rule="self.type == 'LoadBalancerService' || !has(self.loadBalancer)",message="loadBalancer is only valid when type is LoadBalancerService"
// +kubebuilder:validation:XValidation:rule="self.type == 'NodePortService' || !has(self.nodePort)",message="nodePort is only valid when type is NodePortService"
type GatewayEndpointPublishingStrategy struct {
    // type is the publishing strategy.
    //
    // LoadBalancerService: provisions a cloud or hardware LoadBalancer
    // Service. CIO applies platform-specific annotations and manages DNS.
    //
    // NodePortService: provisions a NodePort Service. No DNS is managed.
    // The administrator is responsible for the external load balancer.
    //
    // ClusterIPService: provisions a ClusterIP Service accessible only
    // within the cluster. No DNS is managed. Useful for fronting with
    // an OCP Route on bare-metal clusters.
    //
    // +unionDiscriminator
    // +required
    // +kubebuilder:validation:Enum=LoadBalancerService;NodePortService;ClusterIPService
    Type GatewayEndpointPublishingStrategyType `json:"type"`

    // loadBalancer holds parameters for the LoadBalancer service type.
    // +optional
    LoadBalancer *GatewayLoadBalancerStrategy `json:"loadBalancer,omitempty"`

    // nodePort holds parameters for the NodePort service type.
    // +optional
    NodePort *GatewayNodePortStrategy `json:"nodePort,omitempty"`
}

type GatewayEndpointPublishingStrategyType string

const (
    GatewayStrategyLoadBalancerService GatewayEndpointPublishingStrategyType = "LoadBalancerService"
    GatewayStrategyNodePortService     GatewayEndpointPublishingStrategyType = "NodePortService"
    GatewayStrategyClusterIPService    GatewayEndpointPublishingStrategyType = "ClusterIPService"
)

type GatewayLoadBalancerStrategy struct {
    // scope is External or Internal. External provisions a public-facing
    // LB; Internal provisions a private LB with platform-specific internal
    // annotations (e.g. service.beta.kubernetes.io/aws-load-balancer-internal).
    //
    // +required
    Scope LoadBalancerScope `json:"scope"` // reuses operator/v1 type

    // endpointTrafficPolicy controls how external traffic is routed to
    // proxy pods.
    //
    // Local routes traffic only to proxy pods on the receiving node,
    // preserving source IP and avoiding cross-zone hops. The cloud LB
    // uses healthCheckNodePort; CIO configures this via platform
    // annotations. The OVN local-with-fallback annotation is also set
    // to avoid drops during rolling updates.
    //
    // Cluster routes to any proxy pod (with SNAT). Source IP is not
    // preserved.
    //
    // When omitted, defaults to Local on most platforms. IBM Cloud
    // defaults to Cluster due to platform constraints.
    //
    // +optional
    EndpointTrafficPolicy *GatewayEndpointTrafficPolicy `json:"endpointTrafficPolicy,omitempty"`
}

type GatewayNodePortStrategy struct {
    // endpointTrafficPolicy controls external traffic routing.
    // When Local, the external LB MUST use Service.spec.healthCheckNodePort
    // for health checks or traffic will be dropped on nodes without a
    // local proxy pod.
    //
    // When omitted, the implementation default applies. Unlike
    // LoadBalancerService, there is no platform-specific default.
    //
    // +optional
    EndpointTrafficPolicy *GatewayEndpointTrafficPolicy `json:"endpointTrafficPolicy,omitempty"`
}

// +kubebuilder:validation:Enum=Local;Cluster
type GatewayEndpointTrafficPolicy string

const (
    GatewayEndpointTrafficPolicyLocal   GatewayEndpointTrafficPolicy = "Local"
    GatewayEndpointTrafficPolicyCluster GatewayEndpointTrafficPolicy = "Cluster"
)
```

#### ValidatingAdmissionPolicy

The existing VAP for GatewayClass naming enforces two symmetric rules:

1. A GatewayClass with `controllerName: openshift.io/gateway-controller/v1`
   MUST have a name prefixed with `openshift-`.
2. A GatewayClass with the `openshift-` name prefix MUST use
   `controllerName: openshift.io/gateway-controller/v1`.

The previous hardcoded allowlist of four names is removed. The
`openshift-` prefix is reserved for OpenShift-controller-managed classes.
Unmanaged GatewayClasses (no OpenShift controllerName) may use any name.

#### DNS Management

| `type`               | CIO manages DNS |
|----------------------|-----------------|
| `LoadBalancerService`| Yes             |
| `NodePortService`    | No              |
| `ClusterIPService`   | No              |

### Future API Extensions

The following fields are planned for follow-on EPs and are **not** part
of this EP. They are enumerated here to confirm the `GatewayParameters`
CRD design accommodates them without breaking changes:

- **`spec.resources`** (`corev1.ResourceRequirements`): Configure CPU
  and memory requests/limits for the gateway proxy containers. Translates
  into the `deployment.resources` key in the OSSM defaults ConfigMap.

- **`spec.nodePlacement`**: Node selectors, tolerations, and affinity
  rules for the proxy Deployment. Translates into `deployment.podAnnotations`
  and `deployment.affinity` in the OSSM defaults ConfigMap.

- **Gateway-level reuse**: `GatewayParameters` is designed as a potential
  `gateway.spec.infrastructure.parametersRef` target in a future release,
  enabling per-Gateway overrides without a new CRD. This requires either
  upstream plumbing support or OSSM native support for reading the CRD,
  and is explicitly out of scope for this EP.

### Implementation Details

#### OSSM ConfigMap Mechanism

CIO creates one ConfigMap per GatewayClass in the `openshift-ingress`
namespace, labeled `gateway.istio.io/defaults-for-class: <gatewayclass-name>`.
OSSM reads this label to apply the defaults to all Gateways referencing
that class. This mechanism requires OSSM >= 3.2.4 or >= 3.3.1.

The ConfigMap is owned by CIO and must not be manually edited. Direct
use of this ConfigMap as a customization mechanism by end users is
explicitly unsupported and may be overwritten at any time.

#### Platform Annotation Derivation

CIO reuses its existing IngressController annotation logic, keyed on:

- `scope: External` or `scope: Internal`
- The cluster's infrastructure platform type
- `endpointTrafficPolicy: Local` → OVN
  `traffic-policy.network.alpha.openshift.io/local-with-fallback: ""`
  annotation on applicable platforms

#### Deletion Semantics

When a `GatewayParameters` CR is deleted, CIO removes the defaults
ConfigMap but does not delete the `GatewayClass`. Deleting a GatewayClass
while Gateways reference it would orphan running workloads. CIO sets a
condition on the `GatewayParameters` status and the `GatewayClass`
(`Accepted: False`, reason: `ParametersNotFound`) to notify the
administrator.

#### Feature Gate

Gated behind `GatewayClassParameters` in `TechPreviewNoUpgrade`.

### Risks and Mitigations

**Risk**: OSSM version dependency for the defaults ConfigMap mechanism.

**Mitigation**: CIO checks the OSSM version at reconciliation time and
sets a `Degraded` condition on the `GatewayParameters` status with a
clear message if the version requirement is not met.

**Risk**: `externalTrafficPolicy: Local` with MetalLB BGP can cause
traffic disruption if gateway pods are not spread across all nodes —
MetalLB withdraws BGP route advertisements from nodes without a local
proxy pod, so pod scheduling changes cause route flaps.

**Mitigation**: The field is opt-in. Documentation explicitly covers the
MetalLB interaction and the requirement for even pod distribution.
Future `nodePlacement` support (follow-on) will allow operators to
ensure pods are spread across all BGP-advertising nodes.

**Risk**: ETP Local is not appropriate as a default for existing
GatewayClasses because it could silently break customers who have
been advised by support to rely on ETP Cluster + MetalLB. Changing
an existing GatewayClass default requires a new GatewayClass
(controllerName is immutable).

**Mitigation**: `openshift-default` is unchanged. ETP Local is only
set when explicitly configured in `GatewayParameters`.

## Alternatives

### Extending the `Ingress` Singleton

Adding `spec.gatewayAPI.customGatewayClasses[]` to the existing
`operator.openshift.io/v1alpha1` `Ingress` singleton was considered.
This was rejected because:

- It conflates two concerns: CRD/controller lifecycle management
  (existing `managementMode` field) with GatewayClass infrastructure
  configuration.
- It requires CIO to own and create GatewayClasses on behalf of users,
  rather than users creating their own GatewayClasses with CIO acting
  only as a configuration reconciler.
- The `parametersRef` pattern is the Gateway API's designed extension
  point for this purpose and is already used by other implementations
  (e.g. Envoy Gateway's `EnvoyProxy` CRD).

### Hardcoded GatewayClasses

Creating fixed GatewayClasses (`openshift-external`, `openshift-internal`,
`openshift-clusterip`) was rejected because it requires a code change
for each new configuration combination and creates a permanent VAP
allowlist maintenance burden.

### Istio ClusterIP Alpha Annotation

The annotation `networking.istio.io/service-type: ClusterIP` on a
`Gateway` resource can configure a ClusterIP service today. This is
rejected as a supported path because it is an undocumented private API
that can change or be removed at any OSSM version, gives CIO no
visibility into the configured service type, and does not compose with
ETP configuration.

## Open Questions

1. **`GatewayParameters` status**: What conditions should be reported?
   At minimum: `Accepted` (CIO has found a referencing GatewayClass and
   created the ConfigMap) and `Degraded` (OSSM version too old, ConfigMap
   sync failure).

2. **Default when `endpointPublishingStrategy` is omitted**: Should it
   default to `LoadBalancerService` with `scope: External`, or should
   the field be required?

3. **GatewayClass ownership**: Should CIO require the GatewayClass to
   use the OpenShift controllerName before reconciling a referenced
   `GatewayParameters`, or reconcile for any GatewayClass that points
   to a `GatewayParameters` CR?

## Test Plan

- `[OCPFeatureGate:GatewayClassParameters]` label on all tests
- Unit tests: ConfigMap content derivation, platform annotation selection
- E2E: `ClusterIPService` with OCP Route fronting on bare-metal/vSphere
- E2E: `LoadBalancerService` External/Internal on AWS, Azure, GCP
- E2E: `endpointTrafficPolicy: Local` — source IP preservation, health
  check NodePort set
- Upgrade: `openshift-default` unaffected after upgrade

## Graduation Criteria

### Dev Preview → Tech Preview

- E2E tests on AWS and bare-metal
- OSSM version check implemented
- Documentation draft

### Tech Preview → GA

- E2E on all supported platforms
- Upgrade/downgrade tested
- openshift-docs complete

## Upgrade / Downgrade Strategy

**Upgrade**: No existing GatewayClass or Gateway is modified.
`openshift-default` behavior is unchanged.

**Downgrade**: CIO stops reconciling `GatewayParameters` CRs but does
not delete previously created ConfigMaps or GatewayClasses. Those
resources remain functional until an administrator manually removes them.

## Version Skew Strategy

CIO checks the OSSM version at reconciliation time and reports a
condition rather than creating ConfigMaps an older OSSM cannot consume.

## Infrastructure Needed

None. CIO already has the sail-operator library integration for ConfigMap
management. The new CRD is registered in `openshift/api`.
