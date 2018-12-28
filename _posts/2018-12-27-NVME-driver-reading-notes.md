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
* Construct payload with payload.
* _nvme_ns_cmd_rw() with SPDK_NVME_OPC_READ.
   * nvme_allocate_request()
   * If this controller defines a stripe boundary and this I/O spans a stripe boundary, split the request into multiple requests and submit each separately to hardware.
   * _nvme_ns_cmd_setup_request()
* nvme_qpair_submit_request()
   * If this is a split (parent) request. Submit all of the children but not the parent request itself, since the parent is the original unsplit request.
   * queue those requests which matches with opcode in err_cmd list
   * nvme_transport_qpair_submit_request()
      * nvme_pcie_qpair_submit_request()
      * nvme_rdma_qpair_submit_request()

## 2.5. spdk_nvme_ns_cmd_readv()
```
int spdk_nvme_ns_cmd_readv (struct spdk_nvme_ns * ns,
struct spdk_nvme_qpair * qpair,
uint64_t lba,
uint32_t lba_count,
spdk_nvme_cmd_cb cb_fn,
void * cb_arg,
uint32_t io_flags,
spdk_nvme_req_reset_sgl_cb reset_sgl_fn,
spdk_nvme_req_next_sge_cb next_sge_fn 
)
```
Submit a read I/O to the specified NVMe namespace.
* Construct payload with reset_sgl_fn, next_sge_fn and cb_arg.
* _nvme_ns_cmd_rw() with SPDK_NVME_OPC_READ.
   * nvme_allocate_request()
   * If this controller defines a stripe boundary and this I/O spans a stripe boundary, split the request into multiple requests and submit each separately to hardware.
   * _nvme_ns_cmd_setup_request()
* nvme_qpair_submit_request()
   * If this is a split (parent) request. Submit all of the children but not the parent request itself, since the parent is the original unsplit request.
   * queue those requests which matches with opcode in err_cmd list
   * nvme_transport_qpair_submit_request()
      * nvme_pcie_qpair_submit_request()
      * nvme_rdma_qpair_submit_request()

## 2.6. spdk_nvme_ns_cmd_read_with_md()
```
int spdk_nvme_ns_cmd_read_with_md (struct spdk_nvme_ns * ns,
struct spdk_nvme_qpair * qpair,
void * payload,
void * metadata,
uint64_t lba,
uint32_t lba_count,
spdk_nvme_cmd_cb cb_fn,
void * cb_arg,
uint32_t io_flags,
uint16_t apptag_mask,
uint16_t apptag 
)
```
Submits a read I/O to the specified NVMe namespace.
* Construct payload with payload and metadata.
* _nvme_ns_cmd_rw() with SPDK_NVME_OPC_READ, apptag_mask and apptag.
   * nvme_allocate_request()
   * If this controller defines a stripe boundary and this I/O spans a stripe boundary, split the request into multiple requests and submit each separately to hardware.
   * _nvme_ns_cmd_setup_request()
* nvme_qpair_submit_request()
   * If this is a split (parent) request. Submit all of the children but not the parent request itself, since the parent is the original unsplit request.
   * queue those requests which matches with opcode in err_cmd list
   * nvme_transport_qpair_submit_request()
      * nvme_pcie_qpair_submit_request()
      * nvme_rdma_qpair_submit_request()

## 2.7. spdk_nvme_ns_cmd_write()
```
int spdk_nvme_ns_cmd_write (struct spdk_nvme_ns * ns,
struct spdk_nvme_qpair * qpair,
void * payload,
uint64_t lba,
uint32_t lba_count,
spdk_nvme_cmd_cb cb_fn,
void * cb_arg,
uint32_t io_flags 
)
```
Submit a write I/O to the specified NVMe namespace.
* Construct payload with payload.
* _nvme_ns_cmd_rw() with SPDK_NVME_OPC_WRITE.
   * nvme_allocate_request()
   * If this controller defines a stripe boundary and this I/O spans a stripe boundary, split the request into multiple requests and submit each separately to hardware.
   * _nvme_ns_cmd_setup_request()
* nvme_qpair_submit_request()
   * If this is a split (parent) request. Submit all of the children but not the parent request itself, since the parent is the original unsplit request.
   * queue those requests which matches with opcode in err_cmd list
   * nvme_transport_qpair_submit_request()
      * nvme_pcie_qpair_submit_request()
      * nvme_rdma_qpair_submit_request()

## 2.8. spdk_nvme_ns_cmd_writev()
```
int spdk_nvme_ns_cmd_writev (struct spdk_nvme_ns * ns,
struct spdk_nvme_qpair * qpair,
uint64_t lba,
uint32_t lba_count,
spdk_nvme_cmd_cb cb_fn,
void * cb_arg,
uint32_t io_flags,
spdk_nvme_req_reset_sgl_cb reset_sgl_fn,
spdk_nvme_req_next_sge_cb next_sge_fn 
)
```
Submit a write I/O to the specified NVMe namespace.
* Construct payload with reset_sgl_fn, next_sge_fn and cb_arg.
* _nvme_ns_cmd_rw() with SPDK_NVME_OPC_WRITE.
   * nvme_allocate_request()
   * If this controller defines a stripe boundary and this I/O spans a stripe boundary, split the request into multiple requests and submit each separately to hardware.
   * _nvme_ns_cmd_setup_request()
* nvme_qpair_submit_request()
   * If this is a split (parent) request. Submit all of the children but not the parent request itself, since the parent is the original unsplit request.
   * queue those requests which matches with opcode in err_cmd list
   * nvme_transport_qpair_submit_request()
      * nvme_pcie_qpair_submit_request()
      * nvme_rdma_qpair_submit_request()

## 2.9. spdk_nvme_ns_cmd_write_with_md()
```
int spdk_nvme_ns_cmd_write_with_md (struct spdk_nvme_ns * ns,
struct spdk_nvme_qpair * qpair,
void * payload,
void * metadata,
uint64_t lba,
uint32_t lba_count,
spdk_nvme_cmd_cb cb_fn,
void * cb_arg,
uint32_t io_flags,
uint16_t apptag_mask,
uint16_t apptag 
)
```
Submit a write I/O to the specified NVMe namespace.
* Construct payload with payload and metadata.
* _nvme_ns_cmd_rw() with SPDK_NVME_OPC_WRITE, apptag_mask and apptag.
   * nvme_allocate_request()
   * If this controller defines a stripe boundary and this I/O spans a stripe boundary, split the request into multiple requests and submit each separately to hardware.
   * _nvme_ns_cmd_setup_request()
* nvme_qpair_submit_request()
   * If this is a split (parent) request. Submit all of the children but not the parent request itself, since the parent is the original unsplit request.
   * queue those requests which matches with opcode in err_cmd list
   * nvme_transport_qpair_submit_request()
      * nvme_pcie_qpair_submit_request()
      * nvme_rdma_qpair_submit_request()

## 2.10. spdk_nvme_ns_cmd_write_zeroes()
```
int spdk_nvme_ns_cmd_write_zeroes (struct spdk_nvme_ns * ns,
struct spdk_nvme_qpair * qpair,
uint64_t lba,
uint32_t lba_count,
spdk_nvme_cmd_cb cb_fn,
void * cb_arg,
uint32_t io_flags 
)
```
Submit a write zeroes I/O to the specified NVMe namespace.
* nvme_allocate_request_null()
* Command opcode is SPDK_NVME_OPC_WRITE_ZEROES.
* nvme_qpair_submit_request()
   * If this is a split (parent) request. Submit all of the children but not the parent request itself, since the parent is the original unsplit request.
   * queue those requests which matches with opcode in err_cmd list
   * nvme_transport_qpair_submit_request()
      * nvme_pcie_qpair_submit_request()
      * nvme_rdma_qpair_submit_request()

## 2.11. spdk_nvme_ns_cmd_dataset_management()
```
int spdk_nvme_ns_cmd_dataset_management (struct spdk_nvme_ns * ns,
struct spdk_nvme_qpair * qpair,
uint32_t type,
const struct spdk_nvme_dsm_range * ranges,
uint16_t num_ranges,
spdk_nvme_cmd_cb cb_fn,
void * cb_arg 
)

/** Bit set of attributes for DATASET MANAGEMENT commands. */
enum spdk_nvme_dsm_attribute {
	SPDK_NVME_DSM_ATTR_INTEGRAL_READ		= 0x1,
	SPDK_NVME_DSM_ATTR_INTEGRAL_WRITE		= 0x2,
	SPDK_NVME_DSM_ATTR_DEALLOCATE			= 0x4,
};
```
Submit a data set management request to the specified NVMe namespace.
* nvme_allocate_request_user_copy()
   * Allocate a request as well as a DMA-capable buffer to copy to/from the user's buffer.
* Command opcode is SPDK_NVME_OPC_DATASET_MANAGEMENT.
* nvme_qpair_submit_request()
   * If this is a split (parent) request. Submit all of the children but not the parent request itself, since the parent is the original unsplit request.
   * queue those requests which matches with opcode in err_cmd list
   * nvme_transport_qpair_submit_request()
      * nvme_pcie_qpair_submit_request()
      * nvme_rdma_qpair_submit_request()

## 2.12. spdk_nvme_ns_cmd_flush()
```
int spdk_nvme_ns_cmd_flush (struct spdk_nvme_ns * ns,
struct spdk_nvme_qpair * qpair,
spdk_nvme_cmd_cb cb_fn,
void * cb_arg 
)
```
Submit a flush request to the specified NVMe namespace.
* nvme_allocate_request_null()
* Command opcode is SPDK_NVME_OPC_FLUSH.
* nvme_qpair_submit_request()
   * If this is a split (parent) request. Submit all of the children but not the parent request itself, since the parent is the original unsplit request.
   * queue those requests which matches with opcode in err_cmd list
   * nvme_transport_qpair_submit_request()
      * nvme_pcie_qpair_submit_request()
      * nvme_rdma_qpair_submit_request()

## 2.13. spdk_nvme_qpair_process_completions()
```
int32_t spdk_nvme_qpair_process_completions (struct spdk_nvme_qpair * qpair,
uint32_t max_completions 
)
```
Process any outstanding completions for I/O submitted on a queue pair.
* error injection for those queued error requests.
* nvme_transport_qpair_process_completions()
   * nvme_pcie_qpair_process_completions()
   * nvme_rdma_qpair_process_completions()
* spdk_nvme_ctrlr_free_io_qpair()
   * A request to delete this qpair was made in the context of this completion routine - so it is safe to delete it now.

## 2.14. spdk_nvme_ctrlr_cmd_admin_raw()
```
int spdk_nvme_ctrlr_cmd_admin_raw (struct spdk_nvme_ctrlr * ctrlr,
struct spdk_nvme_cmd * cmd,
void * buf,
uint32_t len,
spdk_nvme_cmd_cb cb_fn,
void * cb_arg 
)
```
Send the given admin command to the NVMe controller.
* nvme_allocate_request_contig()
      * Construct payload with buffer.
      * nvme_allocate_request()
* Memory copy command.
* nvme_ctrlr_submit_admin_request()
      * nvme_qpair_submit_request() with adminq.

## 2.15. spdk_nvme_ctrlr_process_admin_completions()

## 2.16. spdk_nvme_ctrlr_cmd_io_raw()

## 2.17. spdk_nvme_ctrlr_cmd_io_raw_with_md()

