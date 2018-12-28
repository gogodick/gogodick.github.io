---
layout: post
title:  "SPDK NVME driver reading notes"
date:   2018-12-27 12:00:00
categories: SPDK
tags: SPDK NVME
excerpt: Analyze SPDK NVME driver
mathjax: true
---
# 1. Terminology
* NVME

NVM Express (NVMe) or Non-Volatile Memory Host Controller Interface Specification (NVMHCIS) is an open logical device interface specification for accessing non-volatile storage media attached via a PCI Express (PCIe) bus. The acronym NVM stands for non-volatile memory, which is often NAND flash memory that comes in several physical form factors, including solid-state drives (SSDs), PCI Express (PCIe) add-in cards, M.2 cards, and other forms. NVM Express, as a logical device interface, has been designed to capitalize on the low latency and internal parallelism of solid-state storage devices.

* Namespace

An NVMe namespace is a quantity of non-volatile memory (NVM) that can be formatted into logical blocks. Namespaces are used when a storage virtual machine is configured with the NVMe protocol. One or more namespaces are provisioned and connected to an NVMe host. Each namespace can support various block sizes.

* Controller

A PCI Express function that implements NVM Express.

* Queue pair

Submission Queues and Completion Queues.

* Admin QP

Provided for the initial identification and device management commands.

* IO QP

Created by the host using the Admin QP for read and/or write operations. 

# 2. Public Interface
## 2.1. spdk_nvme_probe()
```
int spdk_nvme_probe (const struct spdk_nvme_transport_id * trid,
void * cb_ctx,
spdk_nvme_probe_cb probe_cb,
spdk_nvme_attach_cb attach_cb,
spdk_nvme_remove_cb remove_cb 
)
```
Enumerate the bus indicated by the transport ID and attach the userspace NVMe driver to each device found if desired.
* nvme_driver_init()
   * The primary process will reserve the shared memory and do the initialization. The secondary process will lookup the existing reserved memory.
* If the trtype is PCIe or trid is NULL, this will scan the local PCIe bus. If the trtype is RDMA, the traddr and trsvcid must point at the location of an NVMe-oF discovery service.
* spdk_nvme_probe_internal()
   * nvme_transport_ctrlr_scan()
      * nvme_pcie_ctrlr_scan()
      * nvme_rdma_ctrlr_scan()
   * For secondary process, search attached controller, call attach_cb() if match.
   * For primary process, call nvme_init_controllers()
      * Initialize all new controllers, nvme_ctrlr_process_init()
      * For NVME_CTRLR_STATE_READY, call attach_cb()

## 2.2. spdk_nvme_ctrlr_alloc_io_qpair()
```
struct spdk_nvme_qpair* spdk_nvme_ctrlr_alloc_io_qpair (struct spdk_nvme_ctrlr * ctrlr,
const struct spdk_nvme_io_qpair_opts * opts,
size_t opts_size 
)
```
Allocate an I/O queue pair (submission and completion queue).
* Get options, get controller register, and verify.
* nvme_transport_ctrlr_create_io_qpair()
   * nvme_pcie_ctrlr_create_io_qpair()
   * nvme_rdma_ctrlr_create_io_qpair()
* nvme_ctrlr_proc_add_io_qpair()
   * Insert qpair to process allocated_io_qpairs.

## 2.3. spdk_nvme_ctrlr_get_ns()
```
struct spdk_nvme_ns* spdk_nvme_ctrlr_get_ns (struct spdk_nvme_ctrlr * ctrlr,
uint32_t ns_id 
)
```
Get a handle to a namespace for the given controller.

## 2.4. spdk_nvme_ns_cmd_read()
```
int spdk_nvme_ns_cmd_read (struct spdk_nvme_ns * ns,
struct spdk_nvme_qpair * qpair,
void * 	payload,
uint64_t lba,
uint32_t lba_count,
spdk_nvme_cmd_cb cb_fn,
void * cb_arg,
uint32_t io_flags 
)
```
Submits a read I/O to the specified NVMe namespace.
