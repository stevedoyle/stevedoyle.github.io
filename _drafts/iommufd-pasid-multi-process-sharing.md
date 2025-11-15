---
title: "Sharing Devices Across Processes with iommufd and PASID"
tags: [Linux, IOMMU, iommufd, PASID]
toc: true
---

In my [previous post on IOMMU lessons](2025-11-15-iommu-lessons.md), I discussed challenges with devices using multiple IOMMU groups. Today, I want to explore a different but related problem: how to safely share a single device among multiple user space processes while maintaining isolation and protection.

## The Challenge

Traditional device passthrough using VFIO allows exclusive access to a device from a single process (or VM). If you want multiple processes to access the same device, you have a few options:

1. **Device multiplexing in kernel** - Write a kernel driver that manages sharing, but this pushes complexity into the kernel
2. **User space coordination** - Have processes coordinate through shared memory, but this lacks hardware-enforced isolation
3. **Virtual functions (SR-IOV)** - Use hardware virtualization, but not all devices support this

Modern IOMMU hardware offers a better solution: **Process Address Space ID (PASID)** support combined with the new **iommufd** subsystem.

## What is PASID?

PASID is a PCIe extension that allows DMA transactions from a physical device to be tagged with an identifier. Think of it as a way for a device to say "this DMA operation belongs to process X" rather than just "this DMA operation is from this device."

Key characteristics:

- PASID is a 20-bit identifier (0-1048575), though hardware may support fewer bits
- Each PASID can have its own IOMMU page table (address space)
- Hardware enforces isolation between different PASIDs
- Devices must explicitly support PASID (check PCIe capabilities)

## What is iommufd?

iommufd is the next-generation replacement for VFIO's IOMMU container model, introduced in Linux 6.2. It provides:

- Direct user space access to IOMMU functionality via `/dev/iommu`
- More flexible device sharing models
- PASID support for per-process isolation
- Better performance through optimized page table management
- Support for nested translation (for VMs)

## Architecture Overview

Here's how iommufd with PASID enables multi-process device sharing:

<div class="mermaid">
graph TB
    subgraph "User Space"
        Process1[Process 1<br/>PASID=1]
        Process2[Process 2<br/>PASID=2]
        Process3[Process 3<br/>PASID=3]
    end

    subgraph "Kernel - iommufd"
        iommufd[iommufd<br/>/dev/iommu]
        HWPT1[HWPT 1<br/>Page Tables for PASID 1]
        HWPT2[HWPT 2<br/>Page Tables for PASID 2]
        HWPT3[HWPT 3<br/>Page Tables for PASID 3]
    end

    subgraph "Hardware"
        Device[PASID-Capable Device]
        IOMMU[IOMMU]
        Memory[(System Memory)]
    end

    Process1 --> iommufd
    Process2 --> iommufd
    Process3 --> iommufd

    iommufd --> HWPT1
    iommufd --> HWPT2
    iommufd --> HWPT3

    HWPT1 --> IOMMU
    HWPT2 --> IOMMU
    HWPT3 --> IOMMU

    Device -->|DMA with PASID tag| IOMMU
    IOMMU -->|Translated access| Memory

    style Process1 fill:#e1f5ff
    style Process2 fill:#e1f5ff
    style Process3 fill:#e1f5ff
    style HWPT1 fill:#fff4e1
    style HWPT2 fill:#fff4e1
    style HWPT3 fill:#fff4e1
    style Device fill:#ffe1e1
    style IOMMU fill:#e1ffe1
</div>

Each process gets:
- Its own file descriptor to `/dev/iommu`
- Its own HWPT (Hardware Page Table) with unique PASID
- Hardware-enforced isolation from other processes
- Independent DMA mappings visible only to that PASID

## How It Works

### 1. Device Setup

The device must support PASID. You can check this via the PCIe extended capabilities:

```bash
# Check if device supports PASID
lspci -vvv -s <device> | grep PASID
```

Example output:
```
PASID: Enable+ Exec+ Priv+ PASIDCap: 20
```

### 2. Opening iommufd

Each process opens its own connection to iommufd:

```c
#include <linux/iommufd.h>
#include <sys/ioctl.h>
#include <fcntl.h>

int iommu_fd = open("/dev/iommu", O_RDWR);
if (iommu_fd < 0) {
    perror("Failed to open /dev/iommu");
    return -1;
}
```

### 3. Allocating an IOAS

An IOAS (I/O Address Space) represents the virtual address space visible to the device:

```c
struct iommu_ioas_alloc alloc_cmd = {
    .size = sizeof(alloc_cmd),
    .flags = 0,
};

int ret = ioctl(iommu_fd, IOMMU_IOAS_ALLOC, &alloc_cmd);
if (ret) {
    perror("Failed to allocate IOAS");
    return -1;
}

uint32_t ioas_id = alloc_cmd.out_ioas_id;
```

### 4. Attaching Device with PASID

Here's where PASID support comes in. First, allocate a hardware page table (HWPT) that supports PASID:

```c
struct iommu_hwpt_alloc hwpt_cmd = {
    .size = sizeof(hwpt_cmd),
    .flags = IOMMU_HWPT_ALLOC_PASID,  // Request PASID support
    .dev_id = device_id,
    .pt_id = ioas_id,
    .data_type = IOMMU_HWPT_DATA_NONE,
};

ret = ioctl(iommu_fd, IOMMU_HWPT_ALLOC, &hwpt_cmd);
if (ret) {
    perror("Failed to allocate HWPT with PASID support");
    return -1;
}

uint32_t hwpt_id = hwpt_cmd.out_hwpt_id;
```

Now attach the device with a PASID. The PASID value comes from your device driver or is allocated through a device-specific mechanism (e.g., via a device ioctl that returns an available PASID for your application to use):

```c
// Get PASID from device driver (device-specific mechanism)
uint32_t process_pasid;
ret = ioctl(device_fd, DEVICE_ALLOCATE_PASID, &process_pasid);
if (ret) {
    perror("Failed to allocate PASID from device");
    return -1;
}

// Now attach the HWPT to this PASID
struct iommu_hwpt_attach_pasid attach_cmd = {
    .size = sizeof(attach_cmd),
    .flags = 0,
    .dev_id = device_id,
    .hwpt_id = hwpt_id,
    .pasid = process_pasid,  // PASID allocated by device driver
};

ret = ioctl(iommu_fd, IOMMU_HWPT_ATTACH_PASID, &attach_cmd);
if (ret) {
    perror("Failed to attach PASID to HWPT");
    return -1;
}
```

**Important:** The PASID allocation mechanism is device-specific. Some devices may:

- Allocate PASIDs through a device driver ioctl
- Use a fixed PASID mapping scheme
- Require coordination with a kernel driver for PASID management
- Support PASID allocation via the ENQCMD instruction (on x86)

### 5. Mapping Memory

Each process maps its own memory into its IOAS:

```c
void *buffer = mmap(NULL, buffer_size, PROT_READ | PROT_WRITE,
                    MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

struct iommu_ioas_map map_cmd = {
    .size = sizeof(map_cmd),
    .flags = IOMMU_IOAS_MAP_READABLE | IOMMU_IOAS_MAP_WRITEABLE,
    .ioas_id = ioas_id,
    .user_va = (uint64_t)buffer,
    .length = buffer_size,
    .iova = desired_iova,  // Device-visible address
};

ret = ioctl(iommu_fd, IOMMU_IOAS_MAP, &map_cmd);
if (ret) {
    perror("Failed to map memory");
    return -1;
}
```

### 6. Device Usage

When the device performs DMA, it tags transactions with the appropriate PASID:

```c
// Pseudocode - actual interface depends on device
struct dma_request {
    uint64_t iova;       // Address in IOAS
    uint32_t length;
    uint32_t pasid;      // PASID for this operation
    uint32_t flags;
};

// Device uses PASID to tag DMA
dma_request.pasid = process_pasid;
dma_request.iova = mapped_iova;
submit_to_device(&dma_request);
```

The IOMMU intercepts the DMA transaction, looks up the page table associated with the PASID, and translates the IOVA to a physical address. If the PASID tries to access memory not mapped in its page table, the IOMMU blocks the access and can generate a fault.

## Security and Isolation

The key benefit of PASID-based sharing is hardware-enforced isolation:

<div class="mermaid">
graph LR
    subgraph "Process 1 Memory Space"
        P1Mem[Buffer A<br/>PA: 0x1000]
    end

    subgraph "Process 2 Memory Space"
        P2Mem[Buffer B<br/>PA: 0x2000]
    end

    subgraph "Device View - PASID 1"
        IOVA1[IOVA: 0x0000<br/>→ PA: 0x1000]
    end

    subgraph "Device View - PASID 2"
        IOVA2[IOVA: 0x0000<br/>→ PA: 0x2000]
    end

    Device[Device]

    Device -->|DMA with PASID=1| IOVA1
    Device -->|DMA with PASID=2| IOVA2
    IOVA1 --> P1Mem
    IOVA2 --> P2Mem

    style P1Mem fill:#e1f5ff
    style P2Mem fill:#ffe1e1
    style IOVA1 fill:#e1f5ff
    style IOVA2 fill:#ffe1e1
</div>

Each PASID has its own translation table:
- Process 1's buffer is only visible to PASID 1
- Process 2's buffer is only visible to PASID 2
- Same IOVA (0x0000) maps to different physical addresses
- Hardware prevents cross-PASID access

## Real-World Use Cases

### 1. Machine Learning Accelerators

Multiple training jobs can share a single GPU/TPU:
- Each job gets its own PASID
- Model weights and activations are isolated
- Hardware enforces memory protection
- No kernel-level scheduling required

### 2. Network Interface Cards

Modern SmartNICs with PASID support:
- Multiple applications can use RDMA simultaneously
- Each application has isolated queue pairs
- Zero-copy DMA without security risks
- Hardware-enforced flow isolation

### 3. Storage Accelerators

NVMe devices with PASID support:
- Multiple processes can submit I/O directly
- Per-process I/O queues and buffers
- Kernel-bypass I/O with safety
- Reduced context switching overhead

## Limitations and Considerations

### Hardware Requirements

- Device must support PASID (PCIe ATS/PRI extensions)
- IOMMU must support PASID (Intel VT-d 3.0+, ARM SMMUv3)
- Platform BIOS must enable IOMMU PASID support

### PASID Allocation

- PASID is typically a limited resource (often 20 bits)
- Kernel manages PASID allocation across all devices
- Applications don't choose their own PASID values

### Error Handling

PASID-enabled devices can generate page faults:

```c
// Register for page fault notifications
struct iommu_hwpt_set_fault_handler fault_cmd = {
    .size = sizeof(fault_cmd),
    .hwpt_id = hwpt_id,
    .fault_fd = eventfd(0, EFD_NONBLOCK),
};

ret = ioctl(iommu_fd, IOMMU_HWPT_SET_FAULT_HANDLER, &fault_cmd);
```

When a fault occurs, you can:
- Dynamically map additional memory
- Handle sparse mappings on-demand
- Implement user space page fault handling

### Performance

PASID adds overhead:
- Additional tag bits in PCIe transactions
- IOMMU must look up PASID-specific page tables
- Page table cache (IOTLB) pressure increases
- Consider using huge pages to reduce TLB misses

## Comparison with VFIO

| Feature | VFIO Container | iommufd + PASID |
|---------|----------------|-----------------|
| Multi-process sharing | No | Yes |
| Kernel involvement | High | Low |
| Isolation granularity | Device-level | PASID-level |
| Dynamic mapping | Limited | Full support |
| Page fault handling | No | Yes |
| Nested translation | Limited | Full support |

## Example: Simple Multi-Process DMA

Here's a complete minimal example showing two processes sharing a device:

```c
// common.h
#define DEVICE_PATH "/dev/my_pasid_device"
#define BUFFER_SIZE 4096

// process1.c
int main() {
    int iommu_fd = open("/dev/iommu", O_RDWR);

    // Allocate IOAS and HWPT with PASID support
    uint32_t ioas_id = allocate_ioas(iommu_fd);
    uint32_t hwpt_id = allocate_hwpt_with_pasid(iommu_fd, ioas_id);

    // Attach device - kernel assigns PASID automatically
    attach_device_with_pasid(iommu_fd, hwpt_id);

    // Map buffer
    void *buffer = mmap(NULL, BUFFER_SIZE, PROT_READ | PROT_WRITE,
                        MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    map_buffer(iommu_fd, ioas_id, buffer, BUFFER_SIZE, 0x0);

    // Use device with this buffer
    strcpy(buffer, "Hello from Process 1");
    device_dma_write(0x0, strlen(buffer));  // DMA tagged with this process's PASID

    // Cleanup
    cleanup(iommu_fd);
}

// process2.c
int main() {
    // Same steps as process1, but gets different PASID automatically
    int iommu_fd = open("/dev/iommu", O_RDWR);
    uint32_t ioas_id = allocate_ioas(iommu_fd);
    uint32_t hwpt_id = allocate_hwpt_with_pasid(iommu_fd, ioas_id);
    attach_device_with_pasid(iommu_fd, hwpt_id);

    void *buffer = mmap(NULL, BUFFER_SIZE, PROT_READ | PROT_WRITE,
                        MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    map_buffer(iommu_fd, ioas_id, buffer, BUFFER_SIZE, 0x0);

    // Different buffer, same IOVA (0x0), different PASID
    strcpy(buffer, "Hello from Process 2");
    device_dma_write(0x0, strlen(buffer));  // DMA tagged with different PASID

    cleanup(iommu_fd);
}
```

The device receives DMA requests tagged with different PASIDs, and the IOMMU ensures each PASID only accesses its own mapped memory.

## Debugging Tips

### Check PASID Support

```bash
# Check IOMMU driver
dmesg | grep -i "pasid\|iommu"

# Check device capabilities
lspci -vvv -s <device> | grep -A5 "PASID"

# Check iommufd availability
ls -l /dev/iommu
```

### Common Errors

**EOPNOTSUPP**: IOMMU doesn't support PASID
- Verify hardware support in BIOS
- Check kernel config: `CONFIG_IOMMU_SVA`

**EINVAL on attach**: Device not PASID-capable
- Verify PCIe extended capabilities
- Check device driver PASID enablement

**EFAULT on DMA**: PASID accessing unmapped memory
- Check IOAS mappings
- Verify IOVA ranges
- Review fault handler logs

## Conclusion

iommufd with PASID support provides a powerful mechanism for safely sharing devices among multiple user space processes. By leveraging hardware-enforced isolation, you can achieve:

- High-performance device sharing without kernel intervention
- Strong security boundaries between processes
- Flexible memory management with dynamic mapping
- Support for modern use cases like multi-tenant accelerators

The key insight is that PASID extends the isolation boundary from the device level to the process level, enabling new sharing models that weren't possible with traditional VFIO containers.

As more devices and platforms adopt PASID support, this approach will become increasingly important for building efficient, secure, and scalable systems that make full use of hardware acceleration.

## References

- [Linux IOMMUFD Documentation](https://docs.kernel.org/userspace-api/iommufd.html)
- [IOMMUFD PASID Support Patches](https://lwn.net/Articles/945588/)
- [PCIe PASID Specification](https://pcisig.com/)
- [Intel VT-d Specification](https://software.intel.com/content/www/us/en/develop/articles/intel-virtualization-technology-for-directed-io-vt-d-enhancing-intel-platforms-for-efficient-virtualization-of-io-devices.html)
