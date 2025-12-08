---
title: "IOMMU Lessons"
date: 2025-11-15
tags: [Linux, IOMMU]
toc: false
---

I recently had the opportunity to write some Linux kernel drivers for an ARM-based chip. One device within the chip had a hardware arrangement where the physical device had two connections to memory and each connection went through a different SMMU. To complicate things further, one of the device's interfaces appeared to the CPU as a PCIe device and the second interface was presented as a platform device via the device tree. This hardware configuration was intended to provide extra bandwidth since the device was intended to perform DMA operations at high throughput.

## Hardware Configuration

<div class="mermaid">
graph TB
    subgraph "ARM-based Chip"
        Device[Physical Device]
        PCIe[PCIe Interface]
        Platform[Platform Interface]
        SMMU1[SMMU #1]
        SMMU2[SMMU #2]
        CPU[CPU]

        Device --> PCIe
        Device --> Platform
        PCIe -->|DMA Path 1| SMMU1
        Platform -->|DMA Path 2| SMMU2
    end

    Memory[(System Memory)]

    SMMU1 --> Memory
    SMMU2 --> Memory
    CPU --> Memory

    style Device fill:#e1f5ff
    style PCIe fill:#fff4e1
    style Platform fill:#fff4e1
    style SMMU1 fill:#ffe1e1
    style SMMU2 fill:#ffe1e1
    style CPU fill:#e1e1ff
    style Memory fill:#e1ffe1
</div>

From a software perspective, this presented several challenges.

1. The device was intended to be used from user space using the VFIO subsystem.
2. The device must use the same IOMMU mappings regardless of which interface the device uses for DMA.

My initial approach was to use the same IOMMU domain for both the PCIe and platform devices. An IOMMU domain represents a shared address space and an IOMMU group represents an isolation boundary, with typically one IOMMU group per device. By putting the IOMMU group for each device into the same IOMMU domain, they should share the same address space. When used with VFIO, the VFIO container maps to an IOMMU domain, so on paper this should work nicely.

## Initial Approach (Doesn't Work)

<div class="mermaid">
graph TB
    subgraph "User Space"
        App[Application]
        VFIOContainer[VFIO Container]
    end

    subgraph "Kernel Space - Desired But Not Possible"
        IOMMUDomain[Shared IOMMU Domain]
        IOMMUGroup1[IOMMU Group 1<br/>PCIe Device]
        IOMMUGroup2[IOMMU Group 2<br/>Platform Device]
    end

    subgraph "Hardware"
        SMMU1[SMMU #1]
        SMMU2[SMMU #2]
    end

    App --> VFIOContainer
    VFIOContainer --> IOMMUDomain
    IOMMUDomain --> IOMMUGroup1
    IOMMUDomain --> IOMMUGroup2
    IOMMUGroup1 -.->|Connected to| SMMU1
    IOMMUGroup2 -.->|Connected to| SMMU2

    style IOMMUDomain fill:#ffe1e1
    style IOMMUGroup1 fill:#ffe1e1
    style IOMMUGroup2 fill:#ffe1e1
    style SMMU1 fill:#ffcccc
    style SMMU2 fill:#ffcccc
</div>

Then reality hit. The Linux kernel has some constraints around SMMU programming. One key constraint is that all IOMMU groups (devices) must be connected to the _same_ physical SMMU device. In my ARM-based chip, this was not the case and so the elegant approach of leveraging the IOMMU domain infrastructure was not going to work.

## Actual Solution (Works)

<div class="mermaid">
graph TB
    subgraph "User Space"
        App[Application]
        VFIOContainer1[VFIO Container 1]
        VFIOContainer2[VFIO Container 2]
    end

    subgraph "Kernel Space - PCIe Path"
        IOMMUDomain1[IOMMU Domain 1]
        IOMMUGroup1[IOMMU Group 1<br/>PCIe Device]
    end

    subgraph "Kernel Space - Platform Path"
        IOMMUDomain2[IOMMU Domain 2]
        IOMMUGroup2[IOMMU Group 2<br/>Platform Device]
    end

    subgraph "Hardware"
        SMMU1[SMMU #1]
        SMMU2[SMMU #2]
    end

    App --> VFIOContainer1
    App --> VFIOContainer2
    VFIOContainer1 --> IOMMUDomain1
    VFIOContainer2 --> IOMMUDomain2
    IOMMUDomain1 --> IOMMUGroup1
    IOMMUDomain2 --> IOMMUGroup2
    IOMMUGroup1 -->|Connected to| SMMU1
    IOMMUGroup2 -->|Connected to| SMMU2

    style IOMMUDomain1 fill:#e1f5ff
    style IOMMUDomain2 fill:#e1f5ff
    style IOMMUGroup1 fill:#e1f5ff
    style IOMMUGroup2 fill:#e1f5ff
    style VFIOContainer1 fill:#e1ffe1
    style VFIOContainer2 fill:#e1ffe1
</div>

In the end, I had to fall back to using a separate IOMMU domain and hence VFIO container for each device interface and then update my user space driver library to open both containers and map/unmap memory for both devices.

This is yet another reminder that doing non-standard things in hardware to address a hardware problem (DMA bandwidth in this case) only pushes the problem to software.
