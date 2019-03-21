---
layout: post
title:  "Virtio-net support in DPDK"
date:   2019-03-21 12:00:00
categories: DPDK
tags: DPDK virtio
excerpt: Analyze DPDK vhost library
mathjax: true
---
# 1. Feature Bits
Each virtio device offers all the features it understands. During device initialization, the driver reads this and tells the device the subset that it accepts. The only way to renegotiate is to reset the device.
This allows for forwards and backwards compatibility: if the device is enhanced with a new feature bit, older drivers will not write that feature bit back to the device. Similarly, if a driver is enhanced with a feature that the device doesn’t support, it see the new feature is not offered.
```
struct virtio_net {
	/* Frontend (QEMU) memory and memory region information */
	struct rte_vhost_memory	*mem;
	uint64_t		features;
	uint64_t		protocol_features;
	int			vid;
	uint32_t		flags;
	uint16_t		vhost_hlen;
	/* to tell if we need broadcast rarp packet */
	rte_atomic16_t		broadcast_rarp;
	uint32_t		nr_vring;
	int			dequeue_zero_copy;
	struct vhost_virtqueue	*virtqueue[VHOST_MAX_QUEUE_PAIRS * 2];
#define IF_NAME_SZ (PATH_MAX > IFNAMSIZ ? PATH_MAX : IFNAMSIZ)
	char			ifname[IF_NAME_SZ];
	uint64_t		log_size;
	uint64_t		log_base;
	uint64_t		log_addr;
	struct ether_addr	mac;
	uint16_t		mtu;

	struct vhost_device_ops const *notify_ops;

	uint32_t		nr_guest_pages;
	uint32_t		max_guest_pages;
	struct guest_page       *guest_pages;

	int			slave_req_fd;
} __rte_cache_aligned;

/* The feature bitmap for virtio net */
#define VIRTIO_NET_F_CSUM	0	/* Host handles pkts w/ partial csum */
#define VIRTIO_NET_F_GUEST_CSUM	1	/* Guest handles pkts w/ partial csum */
#define VIRTIO_NET_F_MTU	3	/* Initial MTU advice. */
#define VIRTIO_NET_F_MAC	5	/* Host has given MAC address. */
#define VIRTIO_NET_F_GUEST_TSO4	7	/* Guest can handle TSOv4 in. */
#define VIRTIO_NET_F_GUEST_TSO6	8	/* Guest can handle TSOv6 in. */
#define VIRTIO_NET_F_GUEST_ECN	9	/* Guest can handle TSO[6] w/ ECN in. */
#define VIRTIO_NET_F_GUEST_UFO	10	/* Guest can handle UFO in. */
#define VIRTIO_NET_F_HOST_TSO4	11	/* Host can handle TSOv4 in. */
#define VIRTIO_NET_F_HOST_TSO6	12	/* Host can handle TSOv6 in. */
#define VIRTIO_NET_F_HOST_ECN	13	/* Host can handle TSO[6] w/ ECN in. */
#define VIRTIO_NET_F_HOST_UFO	14	/* Host can handle UFO in. */
#define VIRTIO_NET_F_MRG_RXBUF	15	/* Host can merge receive buffers. */
#define VIRTIO_NET_F_STATUS	16	/* virtio_net_config.status available */
#define VIRTIO_NET_F_CTRL_VQ	17	/* Control channel available */
#define VIRTIO_NET_F_CTRL_RX	18	/* Control channel RX mode support */
#define VIRTIO_NET_F_CTRL_VLAN	19	/* Control channel VLAN filtering */
#define VIRTIO_NET_F_CTRL_RX_EXTRA 20	/* Extra RX mode control support */
#define VIRTIO_NET_F_GUEST_ANNOUNCE 21	/* Guest can announce device on the
					 * network */
#define VIRTIO_NET_F_MQ		22	/* Device supports Receive Flow
					 * Steering */
#define VIRTIO_NET_F_CTRL_MAC_ADDR 23	/* Set MAC address */

/* Do we get callbacks when the ring is completely used, even if we've
 * suppressed them? */
#define VIRTIO_F_NOTIFY_ON_EMPTY	24

/* Can the device handle any descriptor layout? */
#define VIRTIO_F_ANY_LAYOUT		27

/* We support indirect buffer descriptors */
#define VIRTIO_RING_F_INDIRECT_DESC	28

#define VIRTIO_F_VERSION_1		32
#define VIRTIO_F_IOMMU_PLATFORM	33
```

# 2. Virtqueues
The memory aligment and size requirements, in bytes, of each part of the virtqueue are summarized in the
following table:

|Virtqueue Part|Alignment|Size|
| ------ | ------ | ------ |
|Descriptor Table|16|16*(Queue Size)|
|Available Ring|2|6 + 2*(Queue Size)|
|Used Ring|4|6 + 8*(Queue Size)|

When the driver wants to send a buffer to the device, it fills in a slot in the descriptor table (or chains several together), and writes the descriptor index into the available ring. It then notifies the device. When the device has finished a buffer, it writes the descriptor index into the used ring, and sends an interrupt.
```
struct vhost_vring_addr {
	unsigned int index;
	/* Option flags. */
	unsigned int flags;
	/* Flag values: */
	/* Whether log address is valid. If set enables logging. */
#define VHOST_VRING_F_LOG 0

	/* Start of array of descriptors (virtually contiguous) */
	uint64_t desc_user_addr;
	/* Used structure address. Must be 32 bit aligned */
	uint64_t used_user_addr;
	/* Available structure address. Must be 16 bit aligned */
	uint64_t avail_user_addr;
	/* Logging support. */
	/* Log writes to used structure, at offset calculated from specified
	 * address. Address must be 32 bit aligned.
	 */
	uint64_t log_guest_addr;
};
```
