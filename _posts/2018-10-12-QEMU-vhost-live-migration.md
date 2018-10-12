---
layout: post
title:  "Investigating live migration for QEMU vhost"
date:   2018-10-12 12:00:00
categories: QEMU
tags: QEMU virtio live-migration
excerpt: Analyze the implementation of vhost live migration mechanism
mathjax: true
---
# 1. Introduction
vhost related code is at hw/virtio.
Now vhost has two kinds of backend: vhost-user and vhost-kernel
```
typedef enum VhostBackendType {
    VHOST_BACKEND_TYPE_NONE = 0,
    VHOST_BACKEND_TYPE_KERNEL = 1,
    VHOST_BACKEND_TYPE_USER = 2,
    VHOST_BACKEND_TYPE_MAX = 3,
} VhostBackendType;
```
And vhost backend needs to provide below ops:
```
typedef struct VhostOps {
    VhostBackendType backend_type;
    vhost_backend_init vhost_backend_init;
    vhost_backend_cleanup vhost_backend_cleanup;
    vhost_backend_memslots_limit vhost_backend_memslots_limit;
    vhost_net_set_backend_op vhost_net_set_backend;
    vhost_net_set_mtu_op vhost_net_set_mtu;
    vhost_scsi_set_endpoint_op vhost_scsi_set_endpoint;
    vhost_scsi_clear_endpoint_op vhost_scsi_clear_endpoint;
    vhost_scsi_get_abi_version_op vhost_scsi_get_abi_version;
    vhost_set_log_base_op vhost_set_log_base;
    vhost_set_mem_table_op vhost_set_mem_table;
    vhost_set_vring_addr_op vhost_set_vring_addr;
    vhost_set_vring_endian_op vhost_set_vring_endian;
    vhost_set_vring_num_op vhost_set_vring_num;
    vhost_set_vring_base_op vhost_set_vring_base;
    vhost_get_vring_base_op vhost_get_vring_base;
    vhost_set_vring_kick_op vhost_set_vring_kick;
    vhost_set_vring_call_op vhost_set_vring_call;
    vhost_set_vring_busyloop_timeout_op vhost_set_vring_busyloop_timeout;
    vhost_set_features_op vhost_set_features;
    vhost_get_features_op vhost_get_features;
    vhost_set_owner_op vhost_set_owner;
    vhost_reset_device_op vhost_reset_device;
    vhost_get_vq_index_op vhost_get_vq_index;
    vhost_set_vring_enable_op vhost_set_vring_enable;
    vhost_requires_shm_log_op vhost_requires_shm_log;
    vhost_migration_done_op vhost_migration_done;
    vhost_backend_can_merge_op vhost_backend_can_merge;
    vhost_vsock_set_guest_cid_op vhost_vsock_set_guest_cid;
    vhost_vsock_set_running_op vhost_vsock_set_running;
    vhost_set_iotlb_callback_op vhost_set_iotlb_callback;
    vhost_send_device_iotlb_msg_op vhost_send_device_iotlb_msg;
    vhost_get_config_op vhost_get_config;
    vhost_set_config_op vhost_set_config;
    vhost_crypto_create_session_op vhost_crypto_create_session;
    vhost_crypto_close_session_op vhost_crypto_close_session;
} VhostOps;
```
Live migration needs to use some of these ops.
# 2. Live migration
* At first, QEMU needs to specify memory address for vhost backend to write dirty page bitmap.

Calling sequence is: vhost_dev_start()->vhost_set_log_base()

And vhost_set_log_base() is vhost backend ops.

* And then, QEMU needs to notify vhost backend to start writing dirty page bitmap.

Calling sequence is: vhost_log_global_start()->vhost_migration_log()->vhost_dev_set_log()->vhost_dev_set_features()->vhost_set_features()

And vhost_set_features() is vhost backend ops.

* At last, QEMU needs to stop vhost backend and synchronize dirty page memory.

Please refer to vhost_dev_stop():
```
/* Host notifiers must be enabled at this point. */
void vhost_dev_stop(struct vhost_dev *hdev, VirtIODevice *vdev)
{
    int i;

    /* should only be called after backend is connected */
    assert(hdev->vhost_ops);

    for (i = 0; i < hdev->nvqs; ++i) {
        vhost_virtqueue_stop(hdev,
                             vdev,
                             hdev->vqs + i,
                             hdev->vq_index + i);
    }

    if (vhost_dev_has_iommu(hdev)) {
        hdev->vhost_ops->vhost_set_iotlb_callback(hdev, false);
        memory_listener_unregister(&hdev->iommu_listener);
    }
    vhost_log_put(hdev, true);
    hdev->started = false;
    hdev->vdev = NULL;
}
```
Calling sequence is: vhost_dev_stop()->vhost_virtqueue_stop()->vhost_get_vring_base()

vhost_get_vring_base() is vhost backend ops, and it's used to notify backend to stop.

And when backend is stopped, vhost_log_put() is used to synchronize dirty page memory.
