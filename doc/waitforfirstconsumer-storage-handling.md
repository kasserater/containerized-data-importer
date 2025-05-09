# Local Storage Placement for VM Disks

This document describes a special handling of PVCs which have StorageClass with `volumeBindingMode` set to `WaitForFirstConsumer`.  

## Problem description

Local Storage PVs are bound to a specific node. With the binding mode of `WaitForFirstConsumer`  
the binding and the provisioning is delayed until a Pod using the PVC is created. That way the Pod's scheduling constraints
are used to select or provision a PV.

CDI uses worker Pod to import/upload/clone data to the PVC. Creating such a worker Pod will trigger binding of the PVC 
on the node where worker Pod is scheduled. Worker Pod might have different constraints than a kubevirt VM. When the VM is 
scheduled on a different node than the PVC it becomes unusable. The problem might be even bigger when a VM has more than one PVC 
managed by CDI.

## Solution: WaitForFirstConsumer mode for DataVolumes

The CDI has a special handling for DataVolumes (DV) that use storage with `WaitForFirstConsumer` mode. 

The DataVolume controller creates the PersistentVolumeClaim (PVC). When the underlying status of the PVC is Pending/WaitForFirstConsumer 
the CDI will **not schedule** any worker pods (import/clone/upload) associated with that PVC. The DataVolume controller sets 
the DV status to a new phase `WaitForFirstConsumer`. This allows the workload itself (ie. kubevirt) 
to detect a DV phase and handle the initial scheduling which causes the PVC to change to a Bound state
(e.g. by creating and exiting a dummy pod with the same resource requirements as the actual workload). 

**NOTE:** The workload should not attempt to use the contents of the DV until CDI has finished the transfer. 

## Forcing immediate binding

Sometimes you will still want a datavolume to be bound immediately. This can be done by adding the annotation:    
`cdi.kubevirt.io/storage.bind.immediate.requested`.    
This will force the scheduling of a CDI worker pod and immediately bind the PVC.

This is useful for use cases that do not require binding to a particular node (like uploading a golden image to the cluster).      

## Configuration

To be fully compatible with any external tools that may already use CDI, this new feature has to be enabled by 
the feature gate: `HonorWaitForFirstConsumer`. In the released `cdi-cr` it is enabled by default. To disable it remove the feature gate in the `CDI` custom resource, under spec.config (see [cdi-config doc](cdi-config.md)).

A Snippet below shows CDI resource with `HonorWaitForFirstConsumer` enabled.
```
apiVersion: cdi.kubevirt.io/v1beta1
kind: CDI
[...]
spec:
  config:
    featureGates:
    - HonorWaitForFirstConsumer
[...]
```
