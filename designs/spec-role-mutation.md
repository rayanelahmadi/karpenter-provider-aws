# EC2NodeClass spec.role Mutability

This document outlines a proposal for making `spec.role` on `EC2NodeClass` resources mutable.

## Background

Currently `spec.role` is immutable. Karpenter manages an instance profile for each EC2NodeClass when the role is specified. The profile is created at reconciliation time and deleted during finalization.

Users have requested that the role be editable after creation so that the node permissions can be updated without recreating the entire EC2NodeClass.

## Goals

* Allow updates to `spec.role` in place.
* Avoid disrupting running instances that still use the old role.
* Automatically clean up instance profiles that are no longer referenced by nodes.

## Proposed Approach

1. **Multiple Instance Profiles**  
   Instead of a single profile per EC2NodeClass, tag each profile with the owning EC2NodeClass name. When the role is updated, create a new instance profile with the new role and tag it accordingly.
2. **Garbage Collection**  
   Periodically list instance profiles owned by the EC2NodeClass and query EC2 for instances using each profile. Profiles with no running instances are deleted.
3. **Finalization**  
   During EC2NodeClass deletion, remove all owned instance profiles once they are unused. The resource finalizer waits until all profiles are cleaned up.

This approach ensures running nodes keep their existing role until they are replaced while enabling the EC2NodeClass to reference a new role for newly launched nodes.

