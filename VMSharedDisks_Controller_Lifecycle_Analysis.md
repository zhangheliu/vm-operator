# VM Operator: VMSharedDisks Controller Lifecycle Analysis

**Document Version**: 1.0  
**Date**: January 2025  
**Author**: AI Assistant  
**Project**: VMware Tanzu VM Operator  

---

## Executive Summary

This document provides a comprehensive analysis of the current state of VM device controller lifecycle when the `VMSharedDisks` feature is enabled in VMware Tanzu VM Operator.

### Key Findings

- **Current State**: Controllers are created in multiple places without updating `VM.Spec.Hardware.*Controllers`
- **Schema Upgrade**: Backfills `VM.Spec.Hardware.*Controllers` from actual VM hardware
- **Virtual Controller Reconciler**: Manages controllers based on `VM.Spec.Hardware` during updates
- **Behavior**: CD-ROM path only assigns to existing controllers when VMSharedDisks is enabled

---

## Table of Contents

1. [Current Architecture Analysis](#current-architecture-analysis)
2. [Controller Lifecycle Workflow](#controller-lifecycle-workflow)
3. [Current Implementation Analysis](#current-implementation-analysis)
4. [Brownfield VM Behavior](#brownfield-vm-behavior)
5. [Current Code State Analysis](#current-code-state-analysis)
6. [Current Testing State](#current-testing-state)
7. [Current State Summary](#current-state-summary)

---


## Current Architecture Analysis

### Controller Creation Points

| Location | Function | What It Does | Updates VM.Spec.Hardware? |
|----------|----------|--------------|---------------------------|
| `pkg/util/ensure_disk_controller.go` | `EnsureDisksHaveControllers` | Creates PCI, SCSI, SATA, NVME controllers | ❌ NO |
| `pkg/util/configspec.go` | `CopyStorageControllersAndDisks` | Copies controllers from images | ❌ NO |
| `pkg/vmconfig/virtualcontroller/virtualcontroller_reconciler.go` | `Reconcile` | Manages controllers from VM.Spec.Hardware | ✅ YES |

---

## Controller Lifecycle Workflow

### Phase 1: New VM Creation

**VM Creation Flow**:
```
User Creates VM YAML
    ↓
VM Provider Creates ConfigSpec
    ↓
CopyStorageControllersAndDisks (copies controllers from image)
    ↓
EnsureDisksHaveControllers (creates missing controllers)
    ↓
VM Created on vSphere (creates controllers)
    ↓
First Reconciliation
    ↓
Schema Upgrade Runs
    ↓
VM.Spec.Hardware Backfilled (controllers now in spec)
```

**Timeline Analysis**:

| Time | What Adds Controllers | Updates VM.Spec.Hardware? | Is Source of Truth? |
|------|----------------------|---------------------------|-------------------|
| T1-T2: VM Creation | CopyStorageControllersAndDisks, EnsureDisksHaveControllers | ❌ NO | ❌ NO |
| T3: vSphere VM | vSphere creates controllers | N/A | ❌ NO (not in K8s yet) |
| T4: Schema Upgrade | reconcileControllers from moVM | ✅ YES | ✅ YES - Backfills from vSphere |
| T5+: Updates | virtualcontroller_reconciler | ✅ YES | ✅ YES |

### Phase 2: Schema Upgrade

**File**: `pkg/providers/vsphere/upgrade/virtualmachine/vm_schema_upgrade.go`

```go
func reconcileControllers(ctx, vm, moVM) {
    if !pkgcfg.FromContext(ctx).Features.VMSharedDisks {
        return // Skip if feature disabled
    }
    
    // Scan actual VM hardware on vSphere
    for i := range moVM.Config.Hardware.Device {
        switch d := moVM.Config.Hardware.Device[i].(type) {
        case *vimtypes.VirtualIDEController:
            reconcileIDEController(ctx, vm, d)
        case *vimtypes.VirtualNVMEController:
            reconcileNVMEController(ctx, vm, d)
        case *vimtypes.VirtualAHCIController:
            reconcileSATAController(ctx, vm, d)
        case vimtypes.BaseVirtualSCSIController:
            reconcileSCSIController(ctx, vm, d)
        }
    }
}
```

**What It Does**:
1. Scans actual VM hardware on vSphere
2. Backfills `VM.Spec.Hardware.*Controllers` with real controller data
3. Backfills volume controller information
4. Runs only once (controlled by annotations)

### Phase 3: Ongoing Updates

**File**: `pkg/vmconfig/virtualcontroller/virtualcontroller_reconciler.go`

```go
func (r reconciler) Reconcile(ctx, k8sClient, vimClient, vm, moVM, configSpec) error {
    // 1. Collect existing controllers from vSphere VM
    existingControllers := collectExistingControllers(moVM)
    
    // 2. Reconcile with VM.Spec.Hardware.*Controllers
    reconcileSCSIControllers(ctx, configSpec, vm.Spec.Hardware, existingControllers)
    reconcileSATAControllers(ctx, configSpec, vm.Spec.Hardware, existingControllers)
    // ... etc
    
    // 3. Add/Edit/Remove controllers based on spec
    return nil
}
```

---

## Current Implementation Analysis

### Schema Upgrade Implementation

**Current State**: Schema upgrade is already implemented and working.

**File**: `pkg/providers/vsphere/upgrade/virtualmachine/vm_schema_upgrade.go`

**What it does**:
1. Scans actual VM hardware on vSphere (`moVM.Config.Hardware.Device`)
2. Backfills `VM.Spec.Hardware.*Controllers` with real controller data
3. Backfills volume controller information (`controllerType`, `controllerBusNumber`, `unitNumber`)
4. Runs only once (controlled by annotations)
5. Only runs when `VMSharedDisks` feature is enabled

---

## Brownfield VM Behavior

### VMs Created Before VMSharedDisks Feature Enabled

**Initial State**:
```yaml
spec:
  hardware: {}  # No controller specs
  volumes:
    - name: disk1
      persistentVolumeClaim:
        claimName: pvc1
        # No controller info
```

**After Schema Upgrade**:
```yaml
spec:
  hardware:
    scsiControllers:
      - busNumber: 0
        type: "pvscsi"
        sharingMode: "None"
  volumes:
    - name: disk1
      persistentVolumeClaim:
        claimName: pvc1
        controllerType: "pvscsi"        # Backfilled
        controllerBusNumber: 0          # Backfilled
        unitNumber: 0                   # Backfilled
```

**When VMSharedDisks Feature is Enabled**:
- Schema upgrade runs automatically on first reconciliation
- All VMs get controller specs populated from actual hardware
- Volume controller information is also backfilled

### New VMs with User-Defined Controllers and Property Mismatches

**Initial State** (User defines controllers with specific properties):
```yaml
spec:
  hardware:
    scsiControllers:
      - busNumber: 0
        type: "pvscsi"
        sharingMode: "None"
```

**Controller Creation Order During VM Creation**:
1. **VM Class ConfigSpec** is created first (if VM Class has ConfigSpec or devices)
2. **Image controllers and disks** are copied via `CopyStorageControllersAndDisks` (appended to ConfigSpec)
3. **`EnsureDisksHaveControllers`** creates missing controllers if disks need them
4. **No conflict detection** - controllers are blindly copied/appended without checking for conflicts
5. **User-defined controllers in `VM.Spec.Hardware` are NOT consulted during creation**

**What Happens When Controllers Have Property Mismatches**:

Controllers are identified by **controllerType** and **BusNumber**. Property mismatches occur when:
- User specifies SCSI controller (busNumber 0, type "pvscsi", sharingMode "None")
- But vSphere VM was created with SCSI controller (busNumber 0, type "pvscsi", sharingMode "Physical")

**During VM Creation**:
- Controllers from VM Class/Image are created on vSphere VM without checking user spec
- Controller properties (SharingMode, Type, PCISlotNumber) are determined by Class/Image, not user spec
- User-defined controllers in `VM.Spec.Hardware` are ignored during creation

**After Schema Upgrade**:
- Schema upgrade scans actual vSphere VM hardware
- For each controller on vSphere VM, it checks if a controller with the same `busNumber` already exists in `VM.Spec.Hardware`
- **If controller with same `busNumber` exists in spec**: Schema upgrade **skips it entirely** (logs "Skipping reconciliation...that already exists")
  - Schema upgrade does NOT check controller type or properties (SharingMode, Type)
  - User-defined controller in spec is preserved, even if properties don't match vSphere
  - **Property mismatch persists undetected** - spec may have sharingMode "None" but vSphere has "Physical"
- **If controller on vSphere doesn't exist in spec**: Schema upgrade adds it with properties from actual vSphere hardware
- **If controller in spec doesn't exist on vSphere**: It remains in spec but won't be created until first update

**Property Mismatch Resolution During VM Updates**:

When `virtualcontroller_reconciler` runs (during VM updates, VM must be powered off):
- Reconciles controllers by **controllerType + busNumber + properties**
- Uses `scsiDeviceMatchSpec` which checks: PCI slot number, SharingMode, and Type
- **If controller matches spec**: No changes needed
- **If SharingMode differs**: Controller is **edited** to match spec (SharedBus property updated)
- **If Type differs**: Controller is **recreated** (remove old, add new) - requires VM power off
- **If other properties differ**: Handled on case-by-case basis

**Example Scenario**:

1. User creates VM with spec: `SCSI controller (busNumber 0, type "pvscsi", sharingMode "None")`
2. VM Class/Image creates controller: `SCSI controller (busNumber 0, type "pvscsi", sharingMode "Physical")`
3. Schema upgrade sees busNumber 0 exists in spec → **skips backfill** (property mismatch undetected)
4. After first update: `virtualcontroller_reconciler` detects SharingMode mismatch
5. Controller is **edited** to set SharedBus to "None" to match spec

---

## Current State Summary

### What Works

- ✅ Schema upgrade backfills `VM.Spec.Hardware.*Controllers` from actual VM hardware
- ✅ Virtual controller reconciler manages controllers based on `VM.Spec.Hardware`
- ✅ VMSharedDisks feature flag exists and controls behavior
- ✅ CD-ROM path (`updateCdromDeviceChangesWithSharedDisks`) only assigns to existing controllers (does not create controllers)

### Temporary Inconsistency (Handled by Webhooks)

During VM creation, there is a short period of inconsistent state between the vSphere VM hardware and the K8s VM CR resource:

- `EnsureDisksHaveControllers` and `CopyStorageControllersAndDisks` create controllers on the vSphere VM but do not update `VM.Spec.Hardware.*Controllers` in the K8s CR
- The K8s VM CR does not reflect all controllers from the image and class during this period
- Schema upgrade runs on first reconciliation and backfills all controllers from actual vSphere VM hardware to `VM.Spec.Hardware.*Controllers`
- **Webhooks block changes** to `VM.Spec.Hardware.*Controllers` until schema upgrade completes, preventing inconsistencies
- This temporary gap is acceptable and handled correctly by the existing webhook validation



*This document represents a comprehensive analysis of the current state of VM Operator controller lifecycle when VMSharedDisks feature is enabled.*
