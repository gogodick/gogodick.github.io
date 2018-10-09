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
There is a new device driver at drivers/net/ifc, this device driver supports vDPA framework, and this device shoule be a intel FPGA device.
And we will use this device driver to analyze the implementation of live migration.
## 3.1. Dirty page logging

## 3.2. VRING state report/restore

## 3.3. Kick RARP
