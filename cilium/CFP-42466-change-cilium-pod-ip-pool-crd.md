# CFP-42466: change-multi-ip-pool-crd

**SIG: SIG-NAME** ([View all current SIGs](https://docs.cilium.io/en/stable/community/community/#all-sigs))

**Begin Design Discussion:** 2025-11-04

**Cilium Release:** 1.19

**Authors:** kyounghoonJang <matkimchi_@naver.com>

**Status:** Draft

**Reference** this CFP was translated from Korean to English using AI.

## Summary

The current CiliumPodIPPool CRD represents CIDRs as simple string lists, preventing granular control over allocation.
This proposal introduces a schema enhancement that supports marking specific CIDRs as non-allocatable — addressing issues like orphaned IPs and enabling gradual, safe IP pool decommissioning.

## Motivation

When using Multi-Pool IPAM, operators sometimes need to decommission or replace existing CIDRs.
The only current way to stop IP allocation from a CIDR is to remove it from the pool entirely, which can cause existing pod IPs to become orphaned and trigger warnings in the operator.

To support a two-step migration workflow, we propose enabling CIDRs to be marked as “non-allocatable”:

1. Add a new CIDR and mark the old CIDR as non-allocatable.

2. Drain and migrate workloads to nodes using the new CIDR.

3. Once the old CIDR is no longer in use, remove it from the pool.

This allows for controlled and predictable transitions between IP pools without service disruption.

## Goals
Introduce a new `unallocatable` flag in the `CiliumPodIPPool` CRD to mark specific CIDRs as non-allocatable.

## Proposal

### Overview

Current structure:
```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumPodIPPool
metadata:
  name: green-pool
spec:
  ipv4:
    cidrs:
      - 10.20.0.0/16
      - 10.30.0.0/16
    maskSize: 24
```
Proposed structure:
```yaml
apiVersion: cilium.io/v2
kind: CiliumPodIPPool
metadata:
  name: green-pool
spec:
  ipv4:
    cidrs:
      - cidr: 10.20.0.0/16
        unallocatable: true
      - cidr: 10.30.0.0/16
    maskSize: 24
```

### Implementation Details
- Create a struct to hold CIDR and the Unallocatable Flag.
- Implement logic in the allocator to disallow pool allocation when the Unallocatable flag is set.

## Impacts / Key Questions

### Impact 1 : Breaking Change - Schema Migration

Since the schema has changed from storing CIDRs as simple strings in a slice to using a struct that contains both the CIDR and the Unallocatable flag, existing v2alpha users will encounter a schema mismatch issue.

#### Key Question: A way to apply this feature without affecting existing users

To make this work, a Conversion Webhook must be implemented to translate API objects in real-time, 
ensuring existing users can upgrade seamlessly.
Developing a webhook to manage the conversion logic between`v2alpha1` ([]string) and `v2` ([]object) will incur engineering costs.

This does not affect users who were using `v2alpha1`.

##### Option 1: Implement this feature while promoting the Multipool IP functionality to `v2`.

###### Pros
- It is possible to promote the API to v2 while introducing the new feature.

###### Cons
- The changes converting to objects would go into the official v2 release without stability verification.
  
  **Note:** To mitigate this concern, we could introduce the `unallocatable` flag in `v2` under a beta designation

##### Option 2: Upgrade to `v2alpha2` first

###### Pros
- Allows testing the object conversion in a pre-release environment.

###### Cons
- Additional maintenance and migration steps are required before the official v2 release.

#### Key Question: How do we handle resource conflicts and precedence during migration?
If both a v2alpha resource and a v2 resource exist with the same name, how should the operator handle this conflict?

It makes sense to prefer v2 resources, as v2 is generally considered the stable version.
For v2alpha resources, there are two possible handling options

##### Option 1: Leave v2alpha objects and log a warning 

###### Pros
- The operator’s conflict-handling logic becomes simpler.
- Since the user’s CRs are not deleted directly, potential accidents or data loss are avoided.

###### Cons
- Stale CRs may remain unattended, which could lead to confusion.

##### Option 2: Delete or deprecate v2alpha objects

###### Pros
- By having the operator automatically delete CRs, unnecessary resources are removed, preventing any potential confusion for users.

###### Cons
- The operator’s implementation could become more complex.
- If there is no guarantee that all information has been accurately migrated to v2, data loss may occur.

#### Key Question: What happens if the user has to downgrade to an older version of Cilium again

A clear warning should be included in the migration documentation, advising users to proceed with caution when upgrading to v2. For downgrade safety, the existing v2alpha1 objects should remain unchanged. Although these objects may become stale, keeping them allows a manual downgrade to remain possible if needed.