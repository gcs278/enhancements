---
title: aws-lb-subnet-selection
authors:
- "@gcs278"
reviewers:
- "@candita"
- "@frobware"
- "@rfredette"
- "@alebedev87"
- "@miheer"
- "@Miciah"
- "@mtulio"
approvers:
- "@patrickdillon"
api-approvers:
- "@JoelSpeed"
creation-date: 2024-05-29
last-updated: 2024-12-18
tracking-link:
  - https://issues.redhat.com/browse/CORS-3440
see-also:
  - "enhancements/ingress/lb-subnet-selection-aws.md"
replaces:
  - "enhancements/installer/aws-customer-provided-subnets.md"
superseded-by:
---

# Installer LoadBalancer Subnet Selection for AWS

## Summary

This proposal extends the OpenShift Installer's install-config for AWS to enable install-time configuration
of subnets for IngressControllers, including the `default` IngressController and user-created IngressControllers,
as well as the control plane load balancers. It also fixes a common issue where load balancers would
map to unintended subnets ([OCPBUGS-17432](https://issues.redhat.com/browse/OCPBUGS-17432)).

To achieve this, this enhancement deprecates the existing install-config field `platform.aws.subnets`
in favor of a more flexible configuration field that handles specifying subnets for IngressControllers,
control plane load balancers, and the cluster nodes. These new subnet roles, referred to as
IngressControllerLB, ControlPlaneExternalLB, ControlPlaneInternalLB, and ClusterNode subnets
are further defined in [Defining Subnet Roles](#defining-subnets-roles).

## Definitions and Terminology

### The `default` IngressController

This proposal will refer to the IngressController that gets created automatically during installation and handles
the platform routes (console, auth, canary, etc.) as the `default` IngressController. Note that
"default IngressController subnets" refers to the default subnets used by all IngressControllers, including
the `default` IngressController. This is different from "`default`-IngressController subnets", which specifically
means the subnets for the `default` IngressController.

### User-Created IngressControllers

A user-created IngressController is a custom IngressController that is created after installation (Day 2).
It represents all IngressControllers except for the `default` IngressController.

### Defining Subnets Roles

With the install-config now able to specify multiple ways a subnet could be used, it's important
to clarify what these uses are. We will define the function, purpose, or use of a subnet as a "role".
We use the concept of roles to recognize that a single subnet can simultaneously fulfill multiple roles.

Let's define the new subnet roles:

**ClusterNode Subnet Role**

Subnets with this role host the cluster nodes, including both control plane and compute nodes.
Additionally, they serve important functions such as locating the VPC for cluster installation
and determining the Availability Zones for the cluster. Generally these are private subnets,

**IngressControllerLB Subnet Role**

Subnets with this role are designated as the default subnets for hosting AWS load balancers created
specifically for IngressControllers. This role does not include load balancers created from generic
LoadBalancer-type Services, but only those from IngressControllers.

**ControlPlaneExternalLB Subnet Role**

Subnets with this role are designated as subnets for the control plane's **external** load balancer
that serves the Kubernetes API server.

**ControlPlaneInternalLB Subnet Role**

Subnets with this role are designated as subnets for the control plane **internal** load balancers
that serves the Kubernetes API server.

_Note: These roles are defined only to establish a common language within the context of the install-config
and may not be applicable to other environments, situations, or APIs._

### Clarifying Subnet Roles Terminology

For simplicity, "IngressControllerLB subnets," "ControlPlaneExternalLB subnets," "ControlPlaneInternalLB subnets"
and "ClusterNode subnets" refer to subnets that fulfill their respective roles, but may also serve other roles.
Additionally, this proposal will use "ControlPlaneLB subnets" to mean both ControlPlaneExternalLB and
ControlPlaneInternalLB subnets; however "ControlPlaneLB" is NOT an official subnet role.

When this proposal refers to a "IngressController subnet", it is generically referring to a subnet that associated with
an IngressController's load balancer, not specifically a subnet that was specified with the IngressControllerLB role
in the install-config, as these can differ. However, this proposal will avoid using the term "cluster subnet",
due to its ambiguity; it is unclear whether it refers to a ClusterNode, IngressControllerLB, or ControlPlaneLB subnet.

## Motivation

Cluster admins using AWS may have dedicated subnets for their load balancers due to
security reasons, architecture, or infrastructure constraints. They may be installing
a cluster into a VPC with multiple Availability Zones (AZs) containing untagged subnets, but
they wish to restrict their cluster to a single AZ.

Currently, cluster admins can configure their IngressController subnets after installation (Day 2) by configuring
the `spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.classicLoadBalancer.subnets`
or `...aws.networkLoadBalancer.subnets` fields (see [LoadBalancer Subnet Selection for AWS](/enhancements/ingress/lb-subnet-selection-aws.md)).
However, there is no way to explicitly configure the ControlPlaneLB subnets. Cluster admins also need a way to
configure both ControlPlaneLB and IngressControllerLB subnets at install time (Day 1) using the install-config.

Configuring subnets at install time is essential for cluster admins that want to
ensure their load balancer subnets are configured properly from the start. Additionally,
Service Delivery has use cases for ROSA Hosted Control Plane (HCP) that require the `default`
IngressController and ControlPlaneLB subnets to be configured during cluster installation.

### AWS Subnet Discovery

AWS subnet discovery is the mechanism used by the AWS Cloud Controller Manager (CCM) to determine the Elastic Load
Balancer (ELB) subnets created for LoadBalancer-type Services. According to [kubernetes/kubernetes#97431](https://github.com/kubernetes/kubernetes/pull/97431),
a subnet will **NOT** be selected if:

- The `kubernetes.io/cluster/<cluster-id>` tag contains the cluster ID of another cluster.
- The load balancer is external (internet facing) and the subnet is private.
- A subnet for the same AZ has already been chosen based on lexicographical order.
- The subnet type is a `wavelength` or `local`.

_Note: Subnet discovery, as defined here, applies only to IngressControllerLB subnets and not to ControlPlaneLB subnets.
While the Cluster API Provider AWS ([CAPA](https://github.com/kubernetes-sigs/cluster-api-provider-aws)) does some subnet
filtering, it relies on the list of [provided subnets](https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/8f46a4d34d18cb090a4b48c9a9260cd3dffd9162/api/v1beta2/network_types.go#L340)
from the openshift-installer. As a result, ControlPlaneLB subnets are NOT impacted by [OCPBUGS-17432](https://issues.redhat.com/browse/OCPBUGS-17432)._

However, the current subnet discovery implementation has some limitations:

- It selects subnets that lack the `kubernetes.io/cluster/<cluster-id>` tag ([OCPBUGS-17432](https://issues.redhat.com/browse/OCPBUGS-17432))
- It doesn't support use cases such as [Dedicated IngressController Subnets](#dedicated-ingresscontroller-subnets-user-story)

The goal of this enhancement is not to completely bypass or replace subnet discovery by hardcoding the
subnet selection. Subnet discovery offers a more flexible solution for dynamic AZ scaling (only for CLBs)
and aligns with standard workflows and expectations for AWS customers. Instead, this proposal's goal is to allow
users to choose between subnet discovery and explicit subnet selection while introducing new guardrails for
subnet discovery.

### User Stories

#### Dedicated Load Balancer Subnets User Story

_"As a cluster admin, I want to install a cluster into an existing VPC and
configure all of my load balancers to use dedicated subnets so that I can fulfill
my organization's requirements for security, observability, and network segmentation."_

Some enterprise customers have strict requirements to isolate subnets used for their
load balancers. They set up a self-managed VPC with dedicated subnets for load balancers
and require OpenShift to be installed into this configuration. Users may need to isolate
ControlPlaneLB subnets from IngressControllerLB subnets, resulting in multiple sets of dedicated subnets.

See [Specifying Dedicated IngressControllerLB Subnets During Installation with a BYO VPC Workflow](#specifying-dedicated-ingresscontroller-role-subnets-during-installation-with-a-byo-vpc-workflow)
for the workflow for this user story.

#### Do Not Use Untagged Subnets User Stories

1. _"As a cluster admin, I want to install a cluster into an existing VPC that contains
other untagged subnets and I want my IngressControllers to be restricted to the cluster
subnets, so that my load balancers are not unnecessarily mapped to an AZ they aren't
intended for."_
2. _"As a cluster admin, I want to install a private cluster into an existing VPC that
contains other untagged subnets and I want my IngressControllers to be restricted to
private subnets, so that it is not exposed to the public Internet."_

When a cluster admin installs a cluster into an existing VPC with additional non-ClusterNode-role
subnets that are selected by [AWS Subnet Discovery](#aws-subnet-discovery), IngressControllers may
inadvertently use these other subnets, which might be allocated to other cluster or purposes
(see related [RFE-2816](https://issues.redhat.com/browse/RFE-2816) and [OCPBUGS-17432](https://issues.redhat.com/browse/OCPBUGS-17432)).
Allowing load balancers to access unintended subnets can lead to bugs and pose security risks.

_Note: ControlPlaneLB subnets are NOT impacted by this user story because they do not use
[AWS Subnet Discovery](#aws-subnet-discovery)._

If explicitly specifying subnets, this use case will be addressed by the
[Setting IngressController Subnets with BYO VPC during Installation Workflow](#setting-ingresscontroller-subnets-with-byo-vpc-during-installation-workflow).
If using subnet discovery, this use case will be automatically resolved, as the [proposal](#proposal)
is to reject untagged subnets in new installations.

#### ROSA Installation User Story

_"As a ROSA Engineer, I want the ability to specify my load balancer subnets
in the install-config so that I can accommodate installer mechanisms (e.g., Hive) that
don’t support customizing installer manifests."_

Though users can currently [customize](https://github.com/openshift/installer/blob/master/docs/user/customization.md#kubernetes-customization-unvalidated)
the `default` IngressController and ControlPlaneLB subnets via installer manifests, this customization is not supported
for ROSA installations (Classic and HCP). This use case drives the need for an install-config API.

### Goals

- Deprecate the `platform.aws.subnets` field in the install-config and add a
  new, more flexible subnet field for subnet-related configuration.
- Enable users to explicitly configure the subnets for the `default` IngressController on AWS
  through the install-config (i.e. IngressControllerLB subnets).
- Enable users to still use AWS Subnet Discovery for IngressControllers.
- Enables users to explicitly configure the subnets for the ControlPlaneExternalLB and ControlPlaneInternalLB subnets.
- Enable automatic subnet role selection (no opinion)
- Prevent AWS Subnet Discovery from selecting subnets that do not belong to the cluster.
- Provide install-time validation for all user-provided subnets roles.
- Maintain support for configuring the `default` IngressController or ControlPlaneLB subnets via customizing the
  installer manifest.

### Non-Goals

- Extend support to platforms other than AWS.
- Support install-time subnet configuration for LoadBalancer-type Services that aren't associated with an IngressController.
- Remove the `platform.aws.subnets` install-config field.
- Support separate configuration of subnets for compute nodes and control plane nodes.
- Combine ControlPlaneExternalLB and ControlPlaneInternalLB subnet roles into a single role.

## Proposal

To enable cluster admins to specify subnets for IngressControllers and ControlPlaneLBs at install time, we will need
to make the following updates:

1. Deprecate the `platform.aws.subnets` field in the install-config and add a new
   `platform.aws.subnetsConfig` field, which allow users to manage subnet roles.
2. Add the `spec.loadBalancer.platform.aws.classicLoadBalancer.subnets` and
   `...aws.networkLoadBalancer.subnets` fields in the ingresses.config.openshift.io
   (Ingress Config) CRD to encode the default subnets for IngressControllers.
3. Reject new BYO VPC installations that contain untagged subnets.

### InstallConfig API Proposal

The proposal to deprecate the `platform.aws.subnets` field in the install-config is driven by
its ambiguity, it does not distinguish between subnet roles. Additionally, `platform.aws.subnets` is a `[]string` and doesn't
allow for future expansion or adaptation for other subnet type, roles, or metadata information. The new 
install-config field that will enable this feature will be `platform.aws.subnetsConfig`.

#### Manual vs. Automatic Role Selection

Automatic role selection is when the installer, CAPA, AWS CCM, or other components automatically determine how the
provided subnets are used. Currently, the installer performs automatic role selection with the `platform.aws.subnets`
field, as it does not provide a way to override these decisions. The new `platform.aws.subnetsConfig` field will
continue to support automatic role selection for users who prefer a standard, opinionated default cluster subnet
config, but only if no roles are explicitly assigned to any subnet. For IngressControllers, using automatic role
selection enables the AWS CCM to use [AWS Subnet Discovery](#aws-subnet-discovery) as it did previously with
`platform.aws.subnets`.

Manual role selection is when the cluster admin assigns roles to the provided subnets, and the installer
explicitly configures the subnets to be used in accordance to the user's role selection. Every subnet must have
a role, and each role must be assigned to at least one subnet.

### Ingress Config API Proposal

The Ingress Config holds the default values that the Ingress Operator will use when creating
user-created IngressControllers. The Ingress Config `Subnets` fields should also
serve as the default when an existing IngressController switches load balancer types (e.g., from `Classic` to `NLB`).
The Ingress Config will be populated by the installer, similar to how the [Allow Users to specify Load Balancer Type during installation for AWS](/enhancements/installer/aws-load-balancer-type.md)
enhancement introduced the `spec.loadBalancer.platform.aws.lbType` field.

### Rejecting Untagged Subnets Proposal

In BYO VPCs, untagged subnets are nuisance as the AWS CCM may select them, leading to various bugs, RFEs, and
support cases. Since this proposal introduces a new subnet install-config field, it's a great opportunity
to introduce a new behavior which will provide a better user experience. If `platform.aws.subnetsConfig`
is specified to use automatic role selection (i.e. subnet discovery), the installer will inspect
the VPC for untagged subnets and reject the installation if any exist. 

### Implementation Details/Notes/Constraints

As mentioned in [the Proposal section](#proposal), the install-config and the installer will be updated
to support propagating explicit subnet selection to IngressControllers and the control plane load balancers.
The Ingress Config and the Ingress Operator will be updated to support propagating default subnets to all
new IngressControllers, including the `default` IngressController.

#### Installer Updates

The `platform.aws.subnets` field in the install-config will be deprecated and replaced with a new
field, `platform.aws.subnetsConfig`. Much like `Subnets`, the new `SubnetsConfig` field indicates
a list of subnets in a pre-existing VPC, but also can optionally override the roles that the subnet will fulfill in the
cluster. Additionally, since it is a struct and not a `[]string`, it provide the ability to be
expanded with additional subnet-related fields in the future.

The following install-config reflect the initial TechPreview release of this feature:

_Note: The install-config does not currently handle CEL validation, but the following suggestions
use CEL simply as guidelines for the desired validation that the installer needs to implement._ 

```go
// Platform stores all the global configuration that all machinesets
// use.
// +kubebuilder:validation:XValidation:rule=`!(has(self.subnets) && has(self.subnetsConfig))`,message="cannot use both subnets and subnetsConfig"
type Platform struct {
    [...]
    // subnets specifies existing subnets (by ID) where cluster
    // resources will be created.  Leave unset to have the installer
    // create subnets in a new VPC on your behalf.
    //
    // Deprecated: Use subnetsConfig
    //
    // +optional
    Subnets []string `json:"subnets,omitempty"`

    // subnetsConfig defines the subnets in an existing VPC  
    // and specifies their intended roles for use by the installer.
    // If no roles are overridden on any subnet, then the subnet roles
	// are decided automatically. In this case, the VPC must not contain
	// any subnets without the kubernetes.io/cluster/<cluster-id> tag.
	//
	// For overridden subnet role selection, each subnet must have at
	// least one assigned role, and all roles (ClusterNode, IngressControllerLB,
	// ControlPlaneExternalLB, ControlPlaneInternalLB) must be assigned to
	// at least one subnet.
	// 
	// The IngressControllerLB role override specifies the subnets used by the
	// default IngressController and serve as the default for user-created
	// IngressControllers.
    // 
    // Leave this field unset to have the installer create subnets
    // in a new VPC on your behalf. subnetsConfig must contain unique IDs 
    // and must not contain more than 10 subnets with the IngressController
    // role.
    // subnetsConfig is only available in TechPreview.
    //
    // +optional
    // +listType=atomic
    // +kubebuilder:validation:XValidation:rule=`self.all(x, self.exists_one(y, x.id == y.id))`,message="subnetsConfig cannot contain duplicate IDs" 
    // +kubebuilder:validation:XValidation:rule=`self.exists(x, x.roles.exists(r, r == 'ClusterNode'))`,message="subnetsConfig must contain at least 1 subnet with the ClusterNode role"
    // +kubebuilder:validation:XValidation:rule=`self.filter(x, x.roles.exists(r, r == 'IngressControllerLB')).size() <= 10`,message="subnetsConfig must contain less than 10 subnets with the IngressControllerLB role"
    SubnetsConfig []SubnetConfig `json:"subnetsConfig,omitempty"`
}

type SubnetConfig struct {
    // id specifies the subnet ID of an existing subnet.
    // The subnet id must start with "subnet-", consist only
    // of alphanumeric characters, and must be exactly 24
    // characters long.
    //
    // +required
    ID AWSSubnetID `json:"id"`

    // rolesOverride manually overrides the roles (aka functions)
	// that the subnet will provide in the cluster. If no roles are
	// specified on any subnet, then the subnet roles are decided
	// automatically. Each role must be unique.
    //
    // +optional
    // +listType=atomic
    // +kubebuilder:validation:XValidation:rule=`self.all(x, self.exists_one(y, x == y))`,message="rolesOverride cannot contain duplicates"
    RolesOverride []SubnetRole `json:"rolesOverride"`
}

// AWSSubnetID is a reference to an AWS subnet ID.
// +kubebuilder:validation:MinLength=24
// +kubebuilder:validation:MaxLength=24
// +kubebuilder:validation:Pattern=`^subnet-[0-9A-Za-z]+$`
type AWSSubnetID string

// SubnetRole specifies the roles (aka functions) that the subnet will provide in the cluster.
type SubnetRole string

const (
    // ClusterNodeSubnetRole specifies subnets that will be used as subnets for the nodes,
    // API load balancers, and for locating the VPC and Availability Zones.
    ClusterNodeSubnetRole SubnetRole = "ClusterNode"

    // IngressControllerSubnetRole specifies subnets used by the default IngressController and
    // serve as the default for user-created IngressControllers.
    IngressControllerSubnetRole SubnetRole = "IngressControllerLB"

    // ControlPlaneExternalLBSubnetRole specifies subnets used by the external control plane
	// load balancer that serves the Kubernetes API server.
    ControlPlaneExternalLBSubnetRole SubnetRole = "ControlPlaneExternalLB" 

    // ControlPlaneInternalLBSubnetRole specifies subnets used by the internal control plane
    // load balancer that serves the Kubernetes API server.
	ControlPlaneInternalLBSubnetRole SubnetRole = "ControlPlaneInternalLB"
)
```

The `SubnetsConfig` field will be introduced under the TechPreview feature gate, and the `Subnets` field
will be deprecated when `SubnetsConfig` graduates to GA.

##### Installer Validation Rules

###### All or Nothing Subnet Roles Selection

This proposed API follows an "all or nothing" approach for roles: either all roles must be explicitly assigned
(manual role selection), or no roles must be assigned (automatic role selection). The installer must not allow a mix of
automatic and manual role assignments, as this could lead to confusing behavior.

###### All Subnets Belong to the Same VPC Validation

The [existing validation](https://github.com/openshift/installer/blob/0d77aa8df5ddc68e926aa11da24a87981021b256/pkg/asset/installconfig/aws/subnet.go#L91)
for the deprecated `Subnets` field ensuring all subnets belong to the same VPC
should apply to all subnets specified in `SubnetsConfig`.

###### Multiple Load Balancer Subnets in the Same AZ Validation

The installer must reject multiple `IngressControllerLB` role subnets in the same AZ as this will be
rejected by the AWS CCM.

###### SubnetsConfig and Subnets Cannot be Specified Together Validation

Since `Subnets` will be deprecated, the installer must not allow `SubnetsConfig` and `Subnets` to both
be specified.

###### Maximum of 10 IngressController Subnets Validation

Since the IngressController's API [only allows 10 subnets](https://github.com/openshift/api/blob/ee5bb7eaf6b6638d4e3b33ba4ff0834212cdb75d/operator/v1/types_ingress.go#L564),
the installer must not allow more than 10 IngressController subnets as well.

###### Consistent Cluster Scope with IngressControllerLB Subnets Validation

The installer must not allow **any** public IngressControllerLB subnets for private clusters (`installconfig.publish: Internal`)
as this is a security risk as discussed in the [Do Not Use Untagged Subnets User Stories](#do-not-use-untagged-subnets-user-stories) section.

Conversely, the installer must now allow **any** private IngressControllerLB subnets for public clusters
(`installconfig.publish: External`) as the public IngressController will not function with private subnets.

###### Consistent Cluster Scope with ControlPlaneLB Subnets Validation

The installer must not allow **any** public subnets to be specified with the ControlPlaneInternalLB role and must
not allow **any** private subnets to be specified with the ControlPlaneExternalLB role.

Private clusters (`installconfig.publish: Internal`) should reject an install-config containing
anyControlPlaneExternalLB roles, as only an internal control plane load balancer will be created.

###### Reject BYO VPC Installations that Contain Untagged Subnets

As described in [Rejecting Untagged Subnets Proposal](#rejecting-untagged-subnets-proposal), the installer
must not allow installation into an existing VPC if it contains subnets lacking the `kubernetes.io/cluster/<cluster-id>`
tag when subnets roles are not specified. The error message should instruct the user to add a
`kubernetes.io/cluster/unmanaged` tag to any subnets they wish to exclude from this cluster installation or
configure the subnets with the Cluster or IngressController role in `platform.aws.subnetsConfig`.

###### Reject IngressControllers or ControlPlaneLB AZs that do not match ClusterNode AZs

When manual role selection is specified, the installer must ensure that the AZs for the IngressController
(defined by the IngressControllerLB subnets) and control plane (defined by the ControlPlaneExternalLB and
ControlPlaneInternal subnets) match the cluster AZs (defined by the ClusterNode subnets). The installer must
reject installations where the AZs are not equal.

AWS load balancers will NOT register a node located in an AZ that is not enabled. As a result, if the
nodes use an AZ that is not a load balancer AZ, the router pod might be scheduled to  node that the load balancer
cannot reach. Conversely, if the load balancer includes an AZ that is not a node AZ, there will be no nodes in that
AZ for the load balancer to register.

##### Installer-Generated Manifests Updates

The installer will apply the specified IngressControllerLB subnets to the `default` IngressController and the
`cluster` Ingress Config manifests within the `generateDefaultIngressController` and `generateClusterConfig`
functions respectively.

Since the `cluster` Ingress Config also defaults the subnets for the `default` IngressController, technically
we don't need to specify them in the `default` IngressController manifest. However, explicitly configuring
the subnets in the `default` IngressController manifest is more verbose and can make debugging installations
more straightforward.

##### Installer ControlPlaneLB Subnets Configuration Updates

The installer will configure the ControlPlaneInternalLB subnets in the `spec.controlPlaneLoadBalancer.subnets`
field of the CAPI AWSCluster object, while the ControlPlaneExternalLB subnets will be set in the
`spec.secondaryControlPlaneLoadBalancer.subnets` field.

##### Installer ClusterNode Subnets Configuration Updates

The installer will configure the ClusterNode subnets in each of the `spec.subnet` fields of the CAPI AWSMachine
objects for the control plane nodes and in the subnet field of the MachineSets for the compute nodes.

##### Installer Subnet Tagging

All public and private subnets specified in the `platform.aws.subnetsConfig` field, regardless of their role, must be
tagged with the `kubernetes.io/cluster/<cluster-id>: shared` tag. Currently, edge subnets are not tagged by the
installer and should continue to be excluded. These tags assist the AWS CCM in identifying which subnets to use
during [AWS Subnet Discovery](#aws-subnet-discovery) for both the cluster being installed and any other clusters
that may be installed within the same VPC.

#### Ingress Config API Updates

The `spec.loadBalancer.platform.aws.classicLoadBalancer.subnets` and `spec.loadBalancer.platform.aws.networkLoadBalancer.subnets`
fields will be added to the Ingress Config to encode the default subnets for user-created IngressControllers. These
Ingress Config fields provide defaulting for all Ingress Controllers, including the `default` IngressController and
user-created IngressControllers.

Regardless of the load balancer type specified in the `platform.aws.lbType` install-config field, the installer will populate both
`networkLoadBalancer` and `classicLoadBalancer` subnet fields with the values provided in the new
`platform.aws.subnetsConfig` install-config field. The `AWSSubnets` struct is duplicated from the IngressController CRD
to maintain API consistency. Although the new `subnetsConfig` install-config field only supports specifying subnets by
ID, the Ingress Config CRD will still support specifying subnets by name for completeness.

```go
// AWSIngressSpec holds the desired state of the Ingress for Amazon Web Services infrastructure provider.
// This only includes fields that can be modified in the cluster.
// +union
type AWSIngressSpec struct {
    [...]
    // classicLoadBalancerParameters holds configuration parameters for an AWS
    // classic load balancer.
    //
    // +optional
    // +openshift:enable:FeatureGate=IngressConfigLBSubnetsAWS
    ClassicLoadBalancerParameters *AWSClassicLoadBalancerParameters `json:"classicLoadBalancer,omitempty"`

    // networkLoadBalancerParameters holds configuration parameters for an AWS
    // network load balancer.
    //
    // +optional
    //+openshift:enable:FeatureGate=IngressConfigLBSubnetsAWS
    NetworkLoadBalancerParameters *AWSNetworkLoadBalancerParameters `json:"networkLoadBalancer,omitempty"`
}

// AWSClassicLoadBalancerParameters holds configuration parameters for an
// AWS Classic load balancer.
type AWSClassicLoadBalancerParameters struct {
    // subnets specifies the default subnets for all IngressControllers.
    // This default will be overridden if subnets are specified on the IngressController.
    // If omitted, IngressController subnets will be automatically discovered
    // based on the kubernetes.io/cluster/<cluster-id> tag, whether it's public
    // or private, and the availability zone.
    //
    // +optional
    Subnets *AWSSubnets `json:"subnets,omitempty"`
}

// AWSNetworkLoadBalancerParameters holds configuration parameters for an
// AWS Network load balancer.
type AWSNetworkLoadBalancerParameters struct {
    // subnets specifies the default subnets for all IngressControllers. // This default will be overridden if subnets are specified on the IngressController.
    // If omitted, IngressController subnets will be automatically discovered
    // based on the kubernetes.io/cluster/<cluster-id> tag, whether it's public
    // or private, and the availability zone.
    //
    // +optional
    Subnets *AWSSubnets `json:"subnets,omitempty"`
}


// AWSSubnets contains a list of references to AWS subnets by
// ID or name.
// +kubebuilder:validation:XValidation:rule=`has(self.ids) && has(self.names) ? size(self.ids + self.names) <= 10 : true`,message="the total number of subnets cannot exceed 10"
// +kubebuilder:validation:XValidation:rule=`has(self.ids) && self.ids.size() > 0 || has(self.names) && self.names.size() > 0`,message="must specify at least 1 subnet name or id"
type AWSSubnets struct {
    // ids specifies a list of AWS subnets by subnet ID.
    // Subnet IDs must start with "subnet-", consist only
    // of alphanumeric characters, must be exactly 24
    // characters long, must be unique, and the total
    // number of subnets specified by ids and names
    // must not exceed 10.
    //
    // +optional
    // +listType=atomic
    // +kubebuilder:validation:XValidation:rule=`self.all(x, self.exists_one(y, x == y))`,message="subnet ids cannot contain duplicates"
    // + Note: Though it may seem redundant, MaxItems is necessary to prevent exceeding of the cost budget for the validation rules.
    // +kubebuilder:validation:MaxItems=10
    IDs []AWSSubnetID `json:"ids,omitempty"`

    // names specifies a list of AWS subnets by subnet name.
    // Subnet names must not start with "subnet-", must not
    // include commas, must be under 256 characters in length,
    // must be unique, and the total number of subnets
    // specified by ids and names must not exceed 10.
    //
    // +optional
    // +listType=atomic
    // +kubebuilder:validation:XValidation:rule=`self.all(x, self.exists_one(y, x == y))`,message="subnet names cannot contain duplicates"
    // + Note: Though it may seem redundant, MaxItems is necessary to prevent exceeding of the cost budget for the validation rules.
    // +kubebuilder:validation:MaxItems=10
    Names []AWSSubnetName `json:"names,omitempty"`
}

// AWSSubnetID is a reference to an AWS subnet ID.
// +kubebuilder:validation:MinLength=24
// +kubebuilder:validation:MaxLength=24
// +kubebuilder:validation:Pattern=`^subnet-[0-9A-Za-z]+$`
type AWSSubnetID string

// AWSSubnetName is a reference to an AWS subnet name.
// +kubebuilder:validation:MinLength=1
// +kubebuilder:validation:MaxLength=256
// +kubebuilder:validation:XValidation:rule=`!self.contains(',')`,message="subnet name cannot contain a comma"
// +kubebuilder:validation:XValidation:rule=`!self.startsWith('subnet-')`,message="subnet name cannot start with 'subnet-'"
type AWSSubnetName string
```

#### Ingress Operator Updates

The Ingress Operator will be updated to consume the Ingress Config's `spec.loadBalancer.platform.aws.classicLoadBalancer.subnets`
or `spec.loadBalancer.platform.aws.networkLoadBalancer.subnets` fields when creating IngressControllers. The
Ingress Config `subnet` field for the corresponding load balancer type will propagate to the IngressController's relevant
`subnets` field as long as `Subnets` haven't been defined on the IngressController (i.e. defaulting).

This logic will NOT follow the existing defaulting pattern where the default gets encoded in the IngressController
status, because the `Subnets` fields on the IngressController status represent the *actual* state. Instead of encoding
the default in the IngressController object, it will be dynamically sourced from the Ingress Config on each reconcile.
This makes the default `Subnets` behavior dynamic: if the subnets in the Ingress Config are updated, the
IngressController using these defaults will also update, triggering a `Progressing` condition to notify users of the
subnet change (described in [Effectuating Subnet Updates](/enhancements/ingress/lb-subnet-selection-aws.md#effectuating-subnet-updates)).

Additionally, when switching load balancer type, the Ingress Operator will apply the default subnets from the
Ingress Config for the new load balancer type if no subnets are specified in the IngressController for that type. This
process is further detailed in the [`Changing IngressController Load Balancer Type with Defaulting Subnets Workflow`](#changing-ingresscontroller-load-balancer-type-with-defaulting-subnets-workflow).

No validation for this new Ingress Config API will be added to the Ingress Operator as a part of this enhancement.

### Workflow Description

#### Specifying Manual Subnet Roles with BYO VPC during Installation Workflow

A cluster-admin may prefer to explicitly set the default subnets for all IngressControllers (including the `default`
IngressController) during installation (Day 1) rather than relying on [AWS Subnet Discovery](#aws-subnet-discovery).
In order to do this, they must assign all the subnet roles:

1. Cluster admin creates an install-config with `platform.aws.subnetsConfig` specified
   with subnets that manually override the subnet roles:
    ```yaml
    apiVersion: v1
    baseDomain: devcluster.openshift.com
    metadata:
      name: my-cluster
    [...]
    platform:
      aws:
        region: us-east-2
        subnetsConfig:
        - id: subnet-0fcf8e0392f0910d0 # public / us-east-2a
          rolesOverride:
          - IngressControllerLB
          - ControlPlaneExternalLB
        - id: subnet-0fcf8e0392f0910d1 # private / us-east-2a
          rolesOverride:
          - ControlPlaneInternalLB
          - ClusterNode
        lbType: Classic
    [...]
    ```
2. The OpenShift Installer uses the subnets in accordance to their specified roles. For IngressControllerLB subnets,
   it populates the both the `default` IngressController and the `cluster` Ingress Config
   manifests with them and installs the cluster into the existing VPC:
   ```yaml
   apiVersion: config.openshift.io/v1
   kind: Ingress
   metadata:
     name: cluster
   spec:
     [...]
     loadBalancer:
       platform:
         aws:
           classicLoadBalancer:
             subnets:
               ids:
               - subnet-0fcf8e0392f0910d0
               - subnet-0fcf8e0392f0910d1
           networkLoadBalancer:
             subnets:
               ids:
               - subnet-0fcf8e0392f0910d0
               - subnet-0fcf8e0392f0910d1
   [...]
   ---
   apiVersion: operator.openshift.io/v1
   kind: IngressController
   metadata:
     name: default
     namespace: openshift-ingress-operator
   spec:
     [...]
     endpointPublishingStrategy:
       type: LoadBalancerService
       loadBalancer:
         scope: External
         providerParameters:
           type: AWS
           aws:
             type: Classic
             classicLoadBalancer:
               subnets:
                 ids:
                 - subnet-0fcf8e0392f0910d0
                 - subnet-0fcf8e0392f0910d1
     [...]
   ```
3. The `default` IngressController is configured with the specified subnets, and user-created
   IngressControllers created on Day 2 will default to using the subnets specified in the Ingress Config.

#### Using AWS Subnet Discovery for IngressControllers with BYO VPC during Installation Workflow

A cluster admin wants to install a cluster into an existing VPC, but still have the AWS CCM
automatically discover the subnets for its IngressController's load balancer: 

1. Cluster admin creates an install-config with `platform.aws.subnetsConfig` specified
   with the subnet(s) while ensuring `platform.aws.subnetsConfig[].roles` is configured
   with only `ClusterNode`:
    ```yaml
    apiVersion: v1
    baseDomain: devcluster.openshift.com
    metadata:
      name: my-cluster
    [...]
    platform:
      aws:
        region: us-east-2
        subnetsConfig:
        - id: subnet-0fcf8e0392f0910d2
          roles:
          - ClusterNode
        - id: subnet-0fcf8e0392f0910d3
          roles:
          - ClusterNode
    [...]
    ```
2. The OpenShift Installer installs the cluster into the VPC where the ClusterNode subnets exist, but does not
   configure any specific subnets for the `default` IngressController or `cluster` Ingress Config manifests.
3. When the `default` IngressController is created, the load balancer will automatically map to the appropriately
   tagged subnets in the VPC (see [AWS Subnet Discovery](#aws-subnet-discovery) for how subnets are discovered).

#### Specifying Dedicated Load Balancer Subnets During Installation with a BYO VPC Workflow

Specifying dedicated IngressControllerLB or ControlPlaneLB subnets enables cluster admins to isolate
their load balancer subnets from the node subnets (i.e. ClusterNode subnets). Additionally, the
IngressControllerLB and ControlPlaneLB subnets can also be isolated from each other:

1. Cluster admin creates an install-config with `platform.aws.subnetsConfig`, specifying
   each subnet to have a single role (`IngressControllerLB`, `ControlPlaneExternalLB`, `ControlPlaneInternalLB`
   or `ClusterNode`) in the `platform.aws.subnetsConfig.roles` field:
    ```yaml
    apiVersion: v1
    baseDomain: devcluster.openshift.com
    metadata:
      name: my-cluster
    [...]
    platform:
      aws:
        region: us-east-2
        subnetsConfig:
        - id: subnet-0fcf8e0392f0910d4 # Private / AZ us-east-1a
          roles:
          - ClusterNode
        - id: subnet-0fcf8e0392f0910d5 # Public / AZ us-east-1a
          roles:
          - IngressControllerLB
        - id: subnet-0fcf8e0392f0910d6 # Public / AZ us-east-1a
          roles:
          - ControlPlaneExternalLB
        - id: subnet-0fcf8e0392f0910d7 # Private / AZ us-east-1a
          roles:
          - ControlPlaneInternalLB
        lbType: NLB
    [...]
    ```
2. The OpenShift Installer populates the both the `default` IngressController and the `cluster` Ingress Config
   manifests with the IngressControllerLB subnets and installs the cluster into the existing VPC.
3. The `default` IngressController is configured with the dedicated subnet(s), and user-created
   IngressControllers created on Day 2 will also default to using the dedicated subnets.

#### Specifying Dedicated ControlPlaneLB Subnets During Installation with a BYO VPC Workflow

Specifying dedicated ControlPlaneExternalLB and ControlPlaneInternalLB subnets enables cluster admins to isolate
the ControlPlaneLB subnets from the node subnets (i.e. ClusterNode subnets):

1. Cluster admin creates an install-config with `platform.aws.subnetsConfig`, specifying
   each subnet to have either the `ControlPlaneExternalLB` or `ClusterNode` role in `platform.aws.subnetsConfig.roles`, but
   not both:
    ```yaml
    apiVersion: v1
    baseDomain: devcluster.openshift.com
    metadata:
      name: my-cluster
    [...]
    platform:
      aws:
        region: us-east-2
        subnetsConfig:
        - id: subnet-0fcf8e0392f0910d4 # AZ us-east-1a
          roles:
          - ClusterNode
        - id: subnet-0fcf8e0392f0910d5 # AZ us-east-1a
          roles:
          - IngressControllerLB
        lbType: NLB
    [...]
    ```
2. The OpenShift Installer populates the both the `default` IngressController and the `cluster` Ingress Config
   manifests with the IngressControllerLB subnets and installs the cluster into the existing VPC.
3. The `default` IngressController is configured with the dedicated subnet(s), and user-created
   IngressControllers created on Day 2 will also default to using the dedicated subnets.

#### Updating the Default Subnets for New IngressControllers Workflow

When new IngressControllers are created, they will use the subnets specified at install time
unless otherwise specified. If a cluster admin wants to change this default subnet selection
after installation, they can follow these steps:

1. The cluster admin edits the Ingress Config via `oc edit ingresses.config.openshift.io cluster`.
2. The cluster admin updates the subnets in `spec.loadBalancer.platform.aws.classicLoadBalancer.subnets` and/or
   `spec.loadBalancer.platform.aws.networkLoadBalancer.subnets` to their desired default subnets.
3. Inspect the status of all IngressControllers that use the default subnets for `Progressing=true` conditions and
   follow the instructions to recreate the service or patch the spec with the current subnets.

Now all new IngressController will be created with the updated subnets for the load balancer type specified in the
Ingress Config's subnets.

#### Changing IngressController Load Balancer Type with Defaulting Subnets Workflow

If a cluster admin specifies IngressControllerLB subnets in the install-config, and later changes the
IngressController's load balancer type without providing specific subnets, the new load balancer will
default to using the subnets defined in the Ingress Config:

1. The cluster admin installs a cluster with IngressControllerLB subnets and a Classic load balancer for the
   `default` IngressController:
    ```yaml
    apiVersion: v1
    baseDomain: devcluster.openshift.com
    metadata:
      name: my-cluster
    [...]
    platform:
      aws:
        region: us-east-2
        subnetsConfig:
        - id: subnet-0fcf8e0392f0910d0
          roles:
          - IngressControllerLB
          - ClusterNode
        - id: subnet-0fcf8e0392f0910d1
          roles:
          - IngressControllerLB
          - ClusterNode
        lbType: Classic
    [...]
    ```
2. The OpenShift Installer populates the both the `default` IngressController and the `cluster` Ingress Config
   manifests with the IngressControllerLB subnets and installs the cluster into the existing VPC. The
   `default` IngressController only has subnets configured for `classicLoadBalancer`:
   ```yaml
   apiVersion: operator.openshift.io/v1
   kind: IngressController
   metadata:
     name: default
     namespace: openshift-ingress-operator
   spec:
     [...]
     endpointPublishingStrategy:
       type: LoadBalancerService
       loadBalancer:
         scope: External
         providerParameters:
           type: AWS
           aws:
             type: Classic
             classicLoadBalancer:
               subnets:
                 ids:
                 - subnet-0fcf8e0392f0910d0
                 - subnet-0fcf8e0392f0910d1
     [...]
   ```
3. On Day 2, a cluster admin changes the `default` IngressController's load balancer type from `Classic` to `NLB`,
   without explicitly specifying subnets under the `networkLoadBalancer` field. The Ingress Operator defaults the
   NLB's subnets to those specified in the Ingress Config's `spec.loadBalancer.aws.networkLoadBalancer`. However,
   because the IngressController subnets use dynamic defaulting, they do not appear in the spec, but will appear in the
   status:
   ```yaml
   apiVersion: operator.openshift.io/v1
   kind: IngressController
   metadata:
     name: default
     namespace: openshift-ingress-operator
   spec:
     [...]
     endpointPublishingStrategy:
       type: LoadBalancerService
       loadBalancer:
         scope: External
         providerParameters:
           type: AWS
           aws:
             type: NLB
     [...]
   status:
     [...]
     endpointPublishingStrategy:
       type: LoadBalancerService
       loadBalancer:
         scope: External
         providerParameters:
           type: AWS
           aws:
             type: NLB
             networkLoadBalancer:
               subnets:
                 ids:
                 - subnet-0fcf8e0392f0910d0
                 - subnet-0fcf8e0392f0910d1
   ```

As a result, the new NLB type load balancer for the `default` IngressController receives the default subnets specified
at install time.

#### Specifying Dedicated ControlPlaneLB Subnets with BYO VPC during Installation Workflow

A cluster-admin may prefer to explicitly set the subnets for  during installation (Day 1) rather than relying on [AWS Subnet Discovery](#aws-subnet-discovery):

1. Cluster admin creates an install-config with `platform.aws.subnetsConfig` specified
   with the subnet(s) while ensuring the `platform.aws.subnetsConfig[].roles` field is
   configured with `IngressControllerLB` and `ClusterNode`:
    ```yaml
    apiVersion: v1
    baseDomain: devcluster.openshift.com
    metadata:
      name: my-cluster
    [...]
    platform:
      aws:
        region: us-east-2
        subnetsConfig:
        - id: subnet-0fcf8e0392f0910d0
          roles:
          - IngressControllerLB
          - ClusterNode
        - id: subnet-0fcf8e0392f0910d1
          roles:
          - IngressControllerLB
          - ClusterNode
        lbType: Classic
    [...]
    ```
2. The OpenShift Installer populates the both the `default` IngressController and the `cluster` Ingress Config
   manifests with the IngressControllerLB subnets and installs the cluster into the existing VPC:
   ```yaml
   apiVersion: config.openshift.io/v1
   kind: Ingress
   metadata:
     name: cluster
   spec:
     [...]
     loadBalancer:
       platform:
         aws:
           classicLoadBalancer:
             subnets:
               ids:
               - subnet-0fcf8e0392f0910d0
               - subnet-0fcf8e0392f0910d1
           networkLoadBalancer:
             subnets:
               ids:
               - subnet-0fcf8e0392f0910d0
               - subnet-0fcf8e0392f0910d1
   [...]
   ---
   apiVersion: operator.openshift.io/v1
   kind: IngressController
   metadata:
     name: default
     namespace: openshift-ingress-operator
   spec:
     [...]
     endpointPublishingStrategy:
       type: LoadBalancerService
       loadBalancer:
         scope: External
         providerParameters:
           type: AWS
           aws:
             type: Classic
             classicLoadBalancer:
               subnets:
                 ids:
                 - subnet-0fcf8e0392f0910d0
                 - subnet-0fcf8e0392f0910d1
     [...]
   ```
3. The `default` IngressController is configured with the specified subnets, and user-created
   IngressControllers created on Day 2 will default to using the subnets specified in the Ingress Config.

### API Extensions

This proposal doesn't add any API extensions other than the new field proposed in
[Ingress Config API Updates](#ingress-config-api-updates).

### Topology Considerations

#### Hypershift / Hosted Control Planes

This enhancement is directly enabling the https://issues.redhat.com/browse/XCMSTRAT-545
feature for ROSA Hosted Control Planes.

#### Standalone Clusters

This proposal does not have any special implications for standalone clusters.

#### Single-node Deployments or MicroShift

This proposal does not have any special implications for single-node
deployments or MicroShift.

### Risks and Mitigations

- Risk: Using public IngressControllerLB subnets for private clusters.
  - We will reject private cluster installations using public IngressControllerLB subnets (see
    [Consistent Cluster Scope with IngressController-Role Subnets Validation](#consistent-cluster-scope-with-ingresscontroller-role-subnets-validation))
  - For private cluster installations using subnet discovery, public subnets will be excluded from the 
    `default` IngressControllers, provided the subnets are correctly tagged.

### Drawbacks

- Deprecating `platform.aws.subnets` will be painful for users.
- Distinguishing between BYO subnets/VPC and installer-created subnets/VPC may be confusing if we need
  to expand the `platform.aws.subnetsConfig` field in the future to accommodate installer-created
  subnets configuration.
  - For example, there may be a future desire to specify the IngressController role for installer-created subnets,
    and it is unclear how this API design would accommodate that.
- Defaulting subnets for _all_ IngressControllers and for load balancer type updates is complex and subtle.
  - As described in [Ingress Operator Updates](#ingress-operator-updates), we chose to do dynamic defaulting
    for subnets because we don't have an existing mechanism to default the IngressController's spec yet.
  - Updating the subnets in the Ingress Config also updates the subnets for all IngressControllers without subnets
    specified, potentially resulting in many IngressControllers with the `Progressing` condition to fix.

## Open Questions

- Q: Should the install-config design assume all IngressControllerLB subnets are ClusterNode  subnets too?
  - A: No, we must support dedicated IngressController subnet use cases.
- Q: Should the Ingress Config use a feature gate different from the `IngressControllerLBSubnetsAWS` feature gate
  introduced in [NE-705](https://issues.redhat.com/browse/NE-705)?
  - `IngressControllerLBSubnetsAWS` is already enabled by default.
- Q: Bootstrap node using public subnet?

## Test Plan

### Ingress Operator Testing

An E2E test will be created in the Ingress Operator repo to verify the functionality of the new [Ingress Config API](#Ingress-Config-api-updates).
This test will follow a similar pattern established in the existing [TestAWSLBTypeDefaulting](https://github.com/openshift/cluster-ingress-operator/blob/4e621359cea8ef2ae8497101ee3daf9f474b4b66/test/e2e/operator_test.go#L1368) test.

### Installer Testing

E2E test(s) will also be added to the installer to verify functionality of the new [Installer API](#installer-updates).
These tests, typically written by QE, will follow existing patterns established for testing installer functionality in
the [openshift-tests-private](https://github.com/openshift/release/tree/master/ci-operator/config/openshift/openshift-tests-private)
directory of the openshift/release repo. These installer tests are run as nightly CI jobs.

### Impact of Deprecation on Testing

Commands in the Step-Registry will need to be updated to reflect the install-config
migration from `platform.aws.subnets` to `platform.aws.subnetsConfig` including, but
not limited to:

- [aws-load-balancer-tag-vpc-subnets](https://steps.ci.openshift.org/reference/aws-load-balancer-tag-vpc-subnets)
- [ipi-conf-aws-proxy](https://steps.ci.openshift.org/reference/ipi-conf-aws-proxy).
- [ipi-conf-aws-custom-vpc](https://steps.ci.openshift.org/reference/ipi-conf-aws-custom-vpc)
- [ipi-conf-aws-blackholenetwork](https://steps.ci.openshift.org/reference/ipi-conf-aws-blackholenetwork)
- [ipi-conf-aws-edge-zone](https://steps.ci.openshift.org/reference/ipi-conf-aws-edge-zone)
- [ipi-conf-aws-publicsubnets](https://steps.ci.openshift.org/reference/ipi-conf-aws-publicsubnets)
- [ipi-conf-aws-sharednetwork](https://steps.ci.openshift.org/reference/ipi-conf-aws-sharednetwork)

QE will be notified that `platform.aws.subnets` is being deprecated, so they can adjust their existing tests
accordingly.

## Graduation Criteria

This feature (including the deprecation of `Subnets` field) will initially be
released as Tech Preview only, behind the `TechPreviewNoUpgrade` feature gate.

### Dev Preview -> Tech Preview

N/A. This feature will be introduced as Tech Preview.

### Tech Preview -> GA

The E2E tests should be consistently passing and a PR will be created
to enable the feature gate by default.

### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

## Upgrade / Downgrade Strategy

No upgrade or downgrade concerns because all changes are compatible or in the installer.

## Version Skew Strategy

N/A. This enhancement will only be supported for new installations and therefore has no version skew concerns.

## Operational Aspects of API Extensions

N/A.

## Support Procedures

### Ingress Not Functional After Installing using IngressControllerLB Subnets

If the cluster installation with specified IngressControllerLB subnets was successful, but ingress is not working,
check the AWS CCM logs of the leader CCM pod for errors in provisioning the load balancer:

```bash
oc logs -n openshift-cloud-controller-manager aws-cloud-controller-manager-86b68698cd-wfhgz
```

## Alternatives

### Day 1 `default` IngressController Subnet Selection via Installer Manifests Alternative

Starting with 4.17, cluster admins are can configure the subnets of the `default` IngressController
at install time by using [custom installer manifests](https://github.com/openshift/installer/blob/master/docs/user/customization.md#kubernetes-customization-unvalidated).
Custom installer manifests offer an unvalidated method for adding or customizing Kubernetes objects that are
injected into the cluster. The workflow is as follows:

1. Cluster admin runs the openshift-installer command to create the manifests' directory:
   ```bash
   ./openshift-installer create manifests --dir="cluster-foo"
   ```
2. Cluster admin adds the YAML for the `default` IngressController into the manifests directory:
   ```bash
   cat << EOF > cluster-foo/manifests/default-ingresscontroller.yaml
   apiVersion: operator.openshift.io/v1
   kind: IngressController
   metadata:
     name: default
     namespace: openshift-ingress-operator
   spec:
     replicas: 2
     endpointPublishingStrategy:
       type: LoadBalancerService
       loadBalancer:
         scope: External
         providerParameters:
           type: AWS
           aws:
             type: NLB
             networkLoadBalancer:
               subnets:
                 ids:
                 - subnet-09d17e6367aea1305
   EOF
   ```
3. Cluster admin starts the installation with the custom `default` IngressController manifest:
   ```bash
   ./openshift-installer create cluster --dir="cluster-foo"
   ```
4. The cluster is provisioned with the `default` IngressController created, and the Ingress Operator
   will NOT overwrite or edit the IngressController.
   - NOTE: Customers do NOT have to explicitly provide `spec.domain` in the `default` IngressController
     manifest as the Ingress Operator will assume the default value from the Ingress Config.

However, this alternative has been deemed insufficient because:

- The ROSA installer currently lacks support for custom manifests.
- Custom installer manifests are unvalidated, whereas a new API in install-config will have validation.
- The mechanics of providing a `default` IngressController manifest is complicated and error-prone.

### Override Subnet Discovery

One possible design for the install-config API is to require at least one IngressControllerLB subnet for all
new installations, effectively overriding AWS subnet discovery. This approach would bypass subnet discovery
entirely, as the default subnets would always be explicitly defined in the Ingress Config object.

However, as discussed in [AWS Subnet Discovery](#aws-subnet-discovery), subnet discovery offers a more flexible
solution for dynamic AZ scaling (only for CLBs) and aligns with standard workflows and expectations for AWS customers.
After extensive discussions, we have decided to maintain a path for users to continue use subnet discovery.

### Role First API Design Alternative

Alternatively, we could refactor the API to replace the single list of subnets structs in
`platform.aws.subnetsConfig[]` with separate role-centric lists: `platform.aws.subnetsConfig.clusterSubnets[]`
and `platform.aws.subnetsConfig.ingressControllerSubnets[]`:

```go
// Platform stores all the global configuration that all machinesets
// use.
type Platform struct {
    [...]
    // subnetConfig specifies the subnets configuration for
    // a cluster by specifying lists of subnets organized
    // by their role. Leave unset to have the installer
    // create subnets in a new VPC on your behalf.
    //
    // +optional
    SubnetConfig SubnetConfig `json:"subnetConfig,omitempty"`
}

type SubnetConfig struct {
    // clusterSubnets specifies subnets that will be used as
    // ClusterNode role subnets.
    //
    // +required
    // +listType=atomic
    // +kubebuilder:validation:XValidation:rule=`self.all(x, self.exists_one(y, x == y))`,message="clusterSubnets cannot contain duplicates"
    ClusterSubnets []AWSSubnetID `json:"clusterSubnets,omitempty"`

    // ingressControllerSubnets specifies subnets that will be used as
    // IngressControllerLB subnets.
    //
    // +optional
    // +listType=atomic
    // +kubebuilder:validation:XValidation:rule=`self.all(x, self.exists_one(y, x == y))`,message="ingressControllerSubnets cannot contain duplicates"
    IngressControllerSubnets []AWSSubnetID `json:"ingressControllerSubnets,omitempty"`
}

// AWSSubnetID is a reference to an AWS subnet ID.
// +kubebuilder:validation:MinLength=24
// +kubebuilder:validation:MaxLength=24
// +kubebuilder:validation:Pattern=`^subnet-[0-9A-Za-z]+$`
type AWSSubnetID string
```

## Infrastructure Needed

N/A