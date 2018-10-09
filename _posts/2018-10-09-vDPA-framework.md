---
layout: post
title:  "A digest of DPDK vDPA framework"
date:   2018-10-09 12:00:00
categories: DPDK
tags: DPDK virtio
excerpt: Analyze the DPDK vDPA framework
mathjax: true
---
# 1. Introduction
Intel introduces vDPA (vhost data path acceleration) recently, this technique is used to emulate virtio device for VM, and achieve SR-IOV like performance, and support live migration for stock VM.

Intel has submitted related change to DPDK, and this article will take a digest for DPDK vDPA framework.

# 2. Framework
There are two new files for vDPA at lib/librte_vhost, rte_vdpa.h and vdpa.c.
The structure rte_vdpa_device has two fields, addr is PCI address for now.
```
struct rte_vdpa_device {
	struct rte_vdpa_dev_addr addr;
	struct rte_vdpa_dev_ops *ops;
} __rte_cache_aligned;
```
vDPA device driver needs to support below ops:
```
struct rte_vdpa_dev_ops {
	/* Get capabilities of this device */
	int (*get_queue_num)(int did, uint32_t *queue_num);
	int (*get_features)(int did, uint64_t *features);
	int (*get_protocol_features)(int did, uint64_t *protocol_features);

	/* Driver configure/close the device */
	int (*dev_conf)(int vid);
	int (*dev_close)(int vid);

	/* Enable/disable this vring */
	int (*set_vring_state)(int vid, int vring, int state);

	/* Set features when changed */
	int (*set_features)(int vid);

	/* Destination operations when migration done */
	int (*migration_done)(int vid);

	/* Get the vfio group fd */
	int (*get_vfio_group_fd)(int vid);

	/* Get the vfio device fd */
	int (*get_vfio_device_fd)(int vid);

	/* Get the notify area info of the queue */
	int (*get_notify_area)(int vid, int qid,
			uint64_t *offset, uint64_t *size);

	/* Reserved for future extension */
	void *reserved[5];
};
```
And we can use below APIs to register and manage vDPA device:
```
/* Register a vdpa device, return did if successful, -1 on failure */
int __rte_experimental
rte_vdpa_register_device(struct rte_vdpa_dev_addr *addr,
		struct rte_vdpa_dev_ops *ops);

/* Unregister a vdpa device, return -1 on failure */
int __rte_experimental
rte_vdpa_unregister_device(int did);

/* Find did of a vdpa device, return -1 on failure */
int __rte_experimental
rte_vdpa_find_device_id(struct rte_vdpa_dev_addr *addr);

/* Find a vdpa device based on did */
struct rte_vdpa_device * __rte_experimental
rte_vdpa_get_device(int did);
```
# 3. Live migration
There is a new device driver at drivers/net/ifc, this device driver supports vDPA framework, and this device shoule be an intel FPGA device.
And we will use this device driver to analyze the implementation of live migration.
ifc driver supports below ops:
```
struct rte_vdpa_dev_ops ifcvf_ops = {
	.get_queue_num = ifcvf_get_queue_num,
	.get_features = ifcvf_get_vdpa_features,
	.get_protocol_features = ifcvf_get_protocol_features,
	.dev_conf = ifcvf_dev_config,
	.dev_close = ifcvf_dev_close,
	.set_vring_state = NULL,
	.set_features = ifcvf_set_features,
	.migration_done = NULL,
	.get_vfio_group_fd = ifcvf_get_vfio_group_fd,
	.get_vfio_device_fd = ifcvf_get_vfio_device_fd,
	.get_notify_area = ifcvf_get_notify_area,
};
```
## 3.1. Dirty page logging
* ifc driver uses ifcvf_enable_logging() to start dirty page logging, and calling sequence is: ifcvf_set_features()->ifcvf_enable_logging()
```
	if (RTE_VHOST_NEED_LOG(features)) {
		rte_vhost_get_log_base(vid, &log_base, &log_size);
		rte_vfio_container_dma_map(internal->vfio_container_fd,
				log_base, IFCVF_LOG_BASE, log_size);
		ifcvf_enable_logging(&internal->hw, IFCVF_LOG_BASE, log_size);
	}
```
* ifc driver uses ifcvf_disable_logging() to stop dirty page logging, and calling sequence is: update_datapath()->vdpa_ifcvf_stop()->ifcvf_disable_logging()
```
	if (RTE_VHOST_NEED_LOG(features)) {
		ifcvf_disable_logging(hw);
		rte_vhost_get_log_base(internal->vid, &log_base, &log_size);
		rte_vfio_container_dma_unmap(internal->vfio_container_fd,
				log_base, IFCVF_LOG_BASE, log_size);
		/*
		 * IFCVF marks dirty memory pages for only packet buffer,
		 * SW helps to mark the used ring as dirty after device stops.
		 */
		log_buf = (uint8_t *)(uintptr_t)log_base;
		for (i = 0; i < hw->nr_vring; i++)
			ifcvf_used_ring_log(hw, i, log_buf);
	}
```

Above code shows that, hardware marks dirty memory pages for only packet buffer, and ifcvf_used_ring_log() is used to mark the used ring as dirty after device stops.
## 3.2. VRING state report/restore
* ifc driver uses ifcvf_hw_enable() to restore VRING state and start hardware, and calling sequence is: update_datapath()->vdpa_ifcvf_start()->ifcvf_start_hw()->ifcvf_hw_enable()
* ifc driver uses ifcvf_hw_disable() to stop hardware and read VRING state, and calling sequence is: update_datapath()->vdpa_ifcvf_stop()->ifcvf_stop_hw()->ifcvf_hw_disable()
## 3.3. Kick RARP
ifc driver does not support VHOST_USER_PROTOCOL_F_RARP.
```
#define VDPA_SUPPORTED_PROTOCOL_FEATURES \
		(1ULL << VHOST_USER_PROTOCOL_F_REPLY_ACK | \
		 1ULL << VHOST_USER_PROTOCOL_F_SLAVE_REQ | \
		 1ULL << VHOST_USER_PROTOCOL_F_SLAVE_SEND_FD | \
		 1ULL << VHOST_USER_PROTOCOL_F_HOST_NOTIFIER | \
		 1ULL << VHOST_USER_PROTOCOL_F_LOG_SHMFD)
```
migration_done() is used for destination operations when migration done, but ifc driver does not support this API.
```
IFC VF doesn't support RARP packet generation, virtio frontend supporting VIRTIO_NET_F_GUEST_ANNOUNCE feature can help to do that.
```
We can conclude that ifc device does not support RARP, and we can use VIRTIO_NET_F_GUEST_ANNOUNCE at frontend instead.
