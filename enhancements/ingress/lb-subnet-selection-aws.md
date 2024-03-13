---
title: lb-subnet-selection-aws
authors:
  - "@gcs278"
reviewers:
  - "@candita"
  - "@frobware"
  - "@rfredette"
  - "@alebedev87"
  - "@JoelSpeed, for review regarding CCM"
approvers:
  - "@Miciah"
api-approvers:
  - "@deads2k"
creation-date: 2024-03-06
last-updated: 2024-04-02
tracking-link: # link to the tracking ticket (for example: Jira Feature or Epic ticket) that corresponds to this enhancement
  - https://issues.redhat.com/browse/NE-705
see-also:
replaces:
superseded-by:
---

# LoadBalancer Subnet Selection for AWS

## Summary

This enhancement extends the IngressController API to allow a cluster admin to
specify custom subnets for the load balancer in AWS. By default, AWS auto-discovers
the subnets, with logic to break ties when there are multiple subnets for a particular
availability zone. This enhancement will configure the`service.beta.kubernetes.io/aws-load-balancer-subnets`
annotation on the IngressController's LoadBalancer-type service which will manually
configure the subnets for each availability zone.

In addition, this enhancement will also provide an install-time configuration for Day 1
operation by extending the OpenShift install-config API. The install-config API will
provide the ability to set the default subnets for all IngressControllers, including
configuring the default IngressController's subnets.

## Motivation

Cluster admins on AWS may have dedicated subnets for their load balancers due to
security reasons, architecture, or infrastructure constraints. Currently, cluster
admins can manually add the `service.beta.kubernetes.io/aws-load-balancer-subnets`
annotation to the service. However, this approach is considered bad practice since
the service is managed by the Ingress Operator. Additionally, the annotation must
be added after the LoadBalancer-type service has already been created by the Ingress
Operator. This means that an Ingress Controller may initially be deployed in a broken
state. Cluster admins need an API for the Ingress Controller which will populate this
annotation simultaneously with the creation of LoadBalancer-type Services.
Additionally, cluster admins need an install-time option in order to be able to
successfully launch a cluster that has special requirements around subnets.

### User Stories

#### Day 2 Load Balancer Subnet Selection on AWS

_"As a cluster admin, when configuring an IngressController in AWS, I want to
specify the subnet selection for the LoadBalancer-type Service on Day 2."_

The cluster admin can modify the `spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.subnets`
field on the IngressController to specify a list of subnets, one for each
availability zone, for the LoadBalancer-type Service. This is available as a
Day-2 operation to the cluster admin. Note that this enhancement currently
only allows specifying the subnets for IngressController when creating
the IngressController.

#### Maintaining Upgrade Compatibility on AWS

_"As a cluster admin currently using an unmanaged
service.beta.kubernetes.io/aws-load-balancer-subnets service annotation,
I want to upgrade to an OpenShift version that manages this annotation
while maintaining ingress functionality."_

A cluster admin manually set the `service.beta.kubernetes.io/aws-load-balancer-subnets`
annotation, and now wishes to upgrade to OpenShift v4.y which manages the
annotation. They want to ensure that their LoadBalancer's subnets remain unchanged.

1. Cluster admin confirms that initially `service.beta.kubernetes.io/aws-load-balancer-subnets`
   is set on the LoadBalancer-type service managed by the IngressController.
2. The cluster admin upgrades OpenShift to v4.y and the annotation is not
   changed or removed.

If the cluster admin wants to use `spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.subnets`
to specify the subnets for the IngressController, they must delete and recreate
the IngressController. This is because the `Subnet` field is immutable once
the IngressController is created.

#### Day 1 Load Balancer Subnet Selection on AWS

_"As a cluster admin, when installing a cluster in AWS, I want to specify the
default subnet selection for my Ingress Controllers."_

TODO

#### Installing a Private Cluster with Private Subnets in AWS

_"As a cluster admin, when installing a private cluster in AWS, I want to
specify that the default IngressController should use private subnets, so
that it is not exposed to the public Internet."_

When a cluster admin installs a private cluster into a VPC also containing
public subnets lacking the proper tags, internal LoadBalancer-type services
may inadvertently use both the public and private subnets, potentially causing
a security risk. As a workaround to this type of issue, Cluster Admins can specify
the private subnet list in the installconfig's `platform.aws.ingressSubnets` field
to ensure that the default IngressController exclusively uses the private subnets.

### Goals

- Introduce a new field in the IngressController API for subnet selection in AWS.
- Support an install-time (Day 1) configuration for subnet selection of the default
  ingress controller.
- Support migration from an unmanaged `service.beta.kubernetes.io/aws-load-balancer-subnets`
  annotation to using the new API field.
- Support the new API field for both classic ELBs and NLBs.

### Non-Goals

- Extend support to platforms other than AWS.
- Automatically configure subnets for private clusters.
- Extend the feature to ALBs or the AWS Load Balancer Operator (which already has API for subnet configuration).
- Support changing `Subnets` field after IngressController has been created (i.e. mutability)

## Proposal

### Ingress Controller API Proposal

The `IngressController` API is extended by adding an optional and immutable
parameter `Subnets` of type `[]string` to the `AWSLoadBalancerParameters` struct,
to manage the `service.beta.kubernetes.io/aws-load-balancer-subnets` annotation
on the LoadBalancer-type service.

The `service.beta.kubernetes.io/aws-load-balancer-subnets` annotation is used
for both Classic Load Balancers (CLBs) and Network Load Balancers (NLBs), hence
adding it to `AWSLoadBalancerParameters` enables common configuration for both.

While AWS, GCP, and Azure provide annotations for subnet configuration, AWS's
annotation accepts multiple subnets, whereas GCP and Azure only permit a
single subnet. For this reason, we made this API specific to AWS by adding
the configuration in `AWSLoadBalancerParameters`.

```go
// AWSLoadBalancerParameters provides configuration settings that are
// specific to AWS load balancers.
// +union
type AWSLoadBalancerParameters struct {
  [...]

  // subnets specifies the list of subnets for the load balancer to
  // route traffic to. The values can be either a subnetID or subnetName.
  // Each subnet should be from a different availability zones, otherwise
  // the AWS controller logic will break the tie based on the role tag,
  // the cluster tag, and/or lexicographic order.
  // 
  // When omitted, the subnets will be auto-discovered per availability zone.
  // +optional
  // +kubebuilder:validation:XValidation:rule="self == oldSelf",message="subnets is immutable once set"
  // +openshift:enable:FeatureGate=AWSLoadBalancerSubnetSelection
  Subnets []string `json:"subnets,omitempty"`
}
```

#### Immutability

The `Subnets` API field has been made immutable after IngressController
creation for the following reasons:

1. To mitigate the risks outlined in [Invalid Annotation Values](#invalid-annotation-values)
2. To mitigate upgrade compatibility issues

While the AWS Cloud Controller Manager (CCM) handles invalid annotations by
producing errors and events, it is difficult for the Ingress Operator to
decipher these CCM events and indicate to a user that an IngressController
is in a malfunctioning state (i.e., the service isn't getting reconciled) after
the service has been created.

On the other hand, when the IngressController is created and the LoadBalancer-type
service is not yet provisioned, these same invalid `Subnets` values will
prevent the load balancer from being provisioned. The existing Ingress
Operator logic will clearly indicate to the cluster admin that the Load Balancer
failed to provision via the existing `LoadBalancerReady` status condition. In
addition, the `LoadBalancerReady` condition will include the CCM events that
indicate to the cluster admin that the `Subnets` value is invalid. Therefore,
immutability provides a way for the Ingress Operator to produce a clear
signal to the user that `Subnets` is invalid.

Additionally, immutability prevents upgrade incompatibilities. If `Subnets`
cannot be changed after the IngressController has been created, then we can
leave unmanaged `service.beta.kubernetes.io/aws-load-balancer-subnets`
service annotations as they are (i.e., don't remove). In order for cluster
admins to transition from the unmanaged service annotation to the new `Subnets`
API field, they would need to recreate the IngressController.

If in the future a customer needs the `Subnets` field to mutable, this proposal
can be updated to support that as changing from immutable to mutable is a
compatible API change.

### Install Config Proposal (Day 1)

TODO: Install time configuration proposal

### Workflow Description

#### Day 2 Workflow

Setting the subnets on an IngressController created after installation:

1. A cluster admin, who wants to specify the subnets for a load balancer, creates an IngressController with
  `spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.subnets` specified.
2. The ingress operator reconciles the LoadBalancer Service and sets the
   `service.beta.kubernetes.io/aws-load-balancer-subnets` annotation with the subnets provided by the user.

#### Day 1 Workflow

Setting the subnets for the default IngressController during installation:

TODO: Workflow for install time configuration

### API Extensions

This proposal will modify the `IngressController` API by adding a new
field called `Subnets` of type `[]string` to the `AWSLoadBalancerParameters`
struct type.

TODO: What will install-time option modify?

### Topology Considerations

#### Hypershift / Hosted Control Planes

This enhancement is directly enabling the https://issues.redhat.com/browse/XCMSTRAT-545
feature for ROSA Hosted Control Planes.

#### Standalone Clusters

Standalone Clusters impact TBD

#### Single-node Deployments or MicroShift

This proposal does not have any special implications for single-node
deployments or MicroShift.

### Implementation Details/Notes/Constraints

The Ingress Operator will use value of `spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.subnets`
to set the `service.beta.kubernetes.io/aws-load-balancer-subnets` annotation
on the LoadBalancer-type Service created by the ingress controller.

#### Install Time Configuration

TODO

### Risks and Mitigations

#### Invalid Annotation Values

One type of risk is if the cluster admin provides an invalid
`service.beta.kubernetes.io/aws-load-balancer-subnets` annotation value.
In this situation, the AWS Cloud Controller Manager (CCM) will throw an
error and not reconcile the service. 

This means that if a service is created with an invalid annotation, the load
balancer will not be created. If an existing service has an invalid annotation added,
any future updates to the service will remain unreconciled until the invalid
subnet is fixed.

Adding an invalid annotation to an existing Ingress Controller can lead to confusing
scenarios. Without immediate indication, a cluster administrator might mistakenly
believe that adding the subnet(s) was successful, only to discover that future
updates to the service will not be reconciled.

The following are examples of invalid values that can cause an error in the AWS CCM.

1. Multiple Subnets in Same Availability Zone (AZ)
2. Non-Existent Subnets
3. Replacing All Subnets (disjoint union) in a Classic Load Balancer (CLB)

See [Support Procedures](#support-procedures) for how to debug 1 and 2
as well as [Replacing All Subnets in a CLB](#replacing-all-subnets-in-a-clb) for
more details on #3.

##### Replacing All Subnets in a CLB

If a cluster admin changes the list of subnets that would completely replace
the existing subnets on an existing ingress controller using a CLB, the AWS
CCM will throw an error and not reconcile the service. As outlined in this
[GitHub comment](https://github.com/kubernetes/cloud-provider-aws/issues/249#issuecomment-906595623),
a new set of subnets causes the AWS controller attempts to detach all
existing subnets on CLBs which is not a supported operation.

A cluster admin wanting to replace all subnets would need to delete and
recreate the IngressController. It may not be obvious that to the cluster
admin that they are replacing all subnets because they must be aware of which
subnets are currently used by the load balancer. This includes automatically
selected subnets when NOT using `service.beta.kubernetes.io/aws-load-balancer-subnets`.

**Note**: This situation does NOT apply to NLBs as they CAN change all subnets
in one operation.

To mitigate this, we have chosen to make the `Subnets` API field immutable,
as outlined in [Immutability](#immutability).

#### Upgrade Compatibility Risk

The solution in [Maintaining Upgrade Compatibility](#maintaining-upgrade-compatibility)
presents a couple of risks.

Backporting the `Subnets` field to 4.(y-1) while not making it configurable
may confuse cluster admins using that release. A cluster admin using 4.(y-1).z who
hasn't previously configured the service annotation may mistakenly believe the
backported `Subnets` field changes the service annotation. Additionally,
a cluster admin that has previously configured the service annotation in
4.(y-1).z may also expect the backported `Subnets` field to change the service
annotation.

Furthermore, if a cluster upgrades from a 4.(y-1).z version without the
backported API, an upgrade compatibilty risk exists. To mitigate this risk,
we will block edges from 4.(y-1).z versions to 4.y which don't have
the API backport present.

### Drawbacks

This enhancement brings additional engineering complexity
for upgrade scenarios because cluster admins have previously
been allowed to directly add this annotation on a service.

Debugging invalid subnets will be confusing for cluster
admins and may lead to extra support cases or bugs.

## Open Questions

- Do we need to configure subnets for default ingress controller at cluster installation (i.e. Day1 API)?
  - **Answer**: Yes. Service Delivery needs the ability to pick subnets for the
    default ingress controller during installation.

## Test Plan

Unit tests as well as E2E tests will be added to the Ingress Operator
repository for both the target release (4.y) and for the backport
(4.y-1).

An E2E test for the target release (4.y) will cover the following scenarios:

1. Adding/deleting/updating `Subnets` on the IngressController and
   observing `service.beta.kubernetes.io/aws-load-balancer-subnets` on
   the LoadBalancer-type Service.
2. Adding `service.beta.kubernetes.io/aws-load-balancer-subnets` annotation to
   a LoadBalancer-type Service and observing it being reconciled to the
   value of the IngressController's `Subnets` field.

An E2E tests for the (4.y-1) backport will cover the following scenarios:

1. Adding/deleting/updating `service.beta.kubernetes.io/aws-load-balancer-subnets`
   on the service and observing `Subnets` on the IngressController.
2. Modifying `Subnets` on the IngressController and observing it being reconciled
   to the value of `service.beta.kubernetes.io/aws-load-balancer-subnets`
   on the service.

This enhancement also adds an upgrade test for 4.(y-1) to 4.y in
openshift/origin to make sure that the migration workflow explained
in [Maintaining Upgrade Compatibility](#maintaining-upgrade-compatibility)
above works seamlessly. In detail, it implements
[Test](https://github.com/kubernetes/kubernetes/blob/f4e246bc93ffb68b33ed67c7896c379efa4207e7/test/e2e/upgrades/upgrade.go#L48)
interface that includes the following steps:

1. In `Setup`, create an IngressController that creates a load balancer,
   and set the `service.beta.kubernetes.io/aws-load-balancer-subnets` annotation
   on the resulting LoadBalancer-type service.
2. In `Test`, periodically check the service to verify that nothing updates the annotation
   on the service during the upgrade.
3. In `Test`, verify again when the upgrade is done that nothing has updated the annotation.
4. Then change `Subnets` and verify that the operator has set the `service.beta.kubernetes.io/aws-load-balancer-subnets`
   annotation accordingly.

## Graduation Criteria

This feature will initially be released as Tech Preview only, behind the
`TechPreviewNoUpgrade` feature gate. The e2e tests in openshift/origin
described in [Test Plan](#test-plan) will only be added when graduating
this feature to GA.

### Dev Preview -> Tech Preview

N/A. This feature will be introduced as Tech Preview.

### Tech Preview -> GA

The e2e tests as part of openshift/origin should be consistently passing and
a PR will be created to enable the feature gate by default.

### Removing a deprecated feature

N/A

## Upgrade / Downgrade Strategy

Upgrade strategy has been discussed in [Maintaining Upgrade Compatibility](#maintaining-upgrade-compatibility).
This design supports cluster admins who previously set the `service.beta.kubernetes.io/aws-load-balancer-subnets`
annotation by not impacting ingress controller subnet configuration on cluster upgrade.

Downgrades will be also be handled by the design in [Maintaining Upgrade Compatibility](#maintaining-upgrade-compatibility).
Downgrading to 4.(y-1) will preserve the value configured in the `Subnets` field in the
`service.beta.kubernetes.io/aws-load-balancer-subnets` annotation.

## Version Skew Strategy

N/A

## Operational Aspects of API Extensions

N/A

## Support Procedures

### Multiple Subnets in Same AZ Support Procedure

If a cluster admin has provided two subnets in the same Availability Zone (AZ), as
discussed in [Invalid Annotation Values](#invalid-annotation-values),
the IngressController with have a status with a relevant error:

```yaml
  - lastTransitionTime: "2024-04-02T22:07:59Z"
    message: |-
      The service-controller component is reporting SyncLoadBalancerFailed events like: Error syncing load balancer: failed to ensure load balancer: InvalidConfigurationRequest: ELB cannot be attached to multiple subnets in the same AZ.
      The kube-controller-manager logs may contain more details.
    reason: SyncLoadBalancerFailed
    status: "False"
    type: LoadBalancerReady
```

### Non-Existent Subnet Support Procedure

If a cluster admin has provided a subnet that doesn't exist in the `Subnets` field,
as discussed in [Invalid Annotation Values](#invalid-annotation-values),
the IngressController with have a status with a relevant error:

```yaml
  - lastTransitionTime: "2024-04-02T22:07:59Z"
    message: |-
      The service-controller component is reporting SyncLoadBalancerFailed events like: Error syncing load balancer: failed to ensure load balancer: expected to find 1, but found 0 subnets
      The kube-controller-manager logs may contain more details.
    reason: SyncLoadBalancerFailed
    status: "False"
    type: LoadBalancerReady
```

## Alternatives

#### Supporting Mutability

Originally with this proposal, we were planning to support a mutable
`Subnets` API field. However, as documented in [Immutability](#immutability),
we chose to mitigate risks by using immutability.

To support mutability, we would need to understand these concepts
that were solved by immutability:

1. How is upgrade compatibility maintained?
2. How do we provide a signal to the cluster admin when invalid subnets are provided on existing IngressControllers?

##### Mutability Upgrade Compatibility via Backporting

If the `service.beta.kubernetes.io/aws-load-balancer-subnets` annotation
was previously configured and the cluster is upgraded, the default value
`[]` for `spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.subnets`
would clear the annotation for existing IngressControllers, which would
most likely break cluster ingress.

To resolve this issue, we could introduce the `Subnets` field into the previous
OCP version (i.e. 4.(y-1), where y = target release) as a backport. Along with the API
backport, we could add logic specific to 4.(y-1) in the Ingress Operator to forcibly
set the `Subnets` field as the value of the `service.beta.kubernetes.io/aws-load-balancer-subnets`
annotation on the service. Any updates to the `Subnets` field in 4.(y-1) would
automatically be reconciled to the current value of the annotation.

This method of automatically populating the new `Subnets` field in a previous
version prepares the cluster for upgrade by capturing the subnets that the
cluster admin directly configured on the service. When upgrading from version
4.(y-1) to 4.y, the Ingress Operator would use the value captured in the `Subnets`
field to set the service annotation, and directly configuring the service
annotation will no longer be supported.

##### Mutability Upgrade Compatibility via Status

Alternatively, we could detect when a cluster admin first configures
a value for the new `Subnets` field and set a status field "flag" that
would indicate to the Ingress Operator that it can now fully reconcile the
service annotation.

If the status field "flag" is not set, then the Ingress Operator will
ignore an empty (`[]`) `Subnets` field to maintain compatibility on upgrade.

##### Mutability Upgrade Compatibility via EvaluationConditionsDetected

The approach in [LoadBalancer Allowed Source Ranges](/enhancements/ingress/lb-allowed-source-ranges.md)
enhancement addressed this upgrade compatibility issue with a different
approach. It will never remove the service's `spec.loadBalancerSourceRanges`
field when the IngressController's `spec.endpointPublishingStrategy.loadBalancer.allowedSourceRanges`
field is made empty (`[]`). This leads to a less-than-desirable user experience,
requiring the cluster admin to manually clear the `spec.loadBalancerSourceRanges`
field on the service after previously configuring it.

This problem was intended to be temporary, as a `EvaluationConditionsDetected`
was added to detect when it was an appropriate time for the Ingress Operator
to take full control of the service's `spec.loadBalancerSourceRanges`. The
disadvantage is that this work required additional follow-up work that wasn't
completed due to challenges in tracking and prioritizing it.

##### Invalid Subnets on Existing IngressControllers

As documented in [Immutability](#immutability), if we were to support mutability,
we would need a way for the Ingress Operator to detect LoadBalancer-type Service
failures after an IngressController has been created. This is because
the LoadBalancer is already provisioned and the service doesn't indicate
an issue in its status.

To resolve this, the Ingress Operator could attempt to decipher the events
produced by the AWS CCM. As an example, here are the events produced by:

1. Creating a valid LoadBalancer-type service
2. Adding a non-existent subnet in `service.beta.kubernetes.io/aws-load-balancer-subnets`
3. Removing the invalid `service.beta.kubernetes.io/aws-load-balancer-subnets` so the issue resolves

```text
LAST SEEN   TYPE      REASON                   OBJECT         MESSAGE
2s          Normal    EnsuringLoadBalancer     service/test   Ensuring load balancer
2s          Normal    EnsuredLoadBalancer      service/test   Ensured load balancer
12s         Warning   SyncLoadBalancerFailed   service/test   Error syncing load balancer: failed to ensure load balancer: expected to find 1, but found 0 subnets
```

As you'll notice, there is no event generated to indicate that the service
has recovered from a faulty state. This poses a challenge for the Ingress
Operator in translating this into a status for the cluster admin.

We either need to collaborate with the cloud team to find a solution or
acknowledge and document the fact that cluster administrators can disrupt
their service by applying invalid subnets after an IngressController is created.

## Infrastructure Needed [optional]

N/A
