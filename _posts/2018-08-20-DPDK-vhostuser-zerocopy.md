---
layout: post
title:  "Investigating zero copy for DPDK vhost"
date:   2018-08-20 12:00:00
categories: DPDK
tags: DPDK virtio
excerpt: Analyze the implementation of vhost zero copy mechanism
mathjax: true
---
# 1. Introduction
For real device like Intel nics, PMD driver creates TX descriptor ring and RX decriptor ring, and then hardware writes to and reads from system memory through DMA.

For emulated device like vhost, there's no hardware to perform DMA operation. Instead, a UNIX domain socket based mechanism allows to set up the resources used by a number of Vrings shared between two userspace processes, which will be placed in shared memory. 

![vhost architecture](https://raw.githubusercontent.com/gogodick/gogodick.github.io/master/img/vhost_architecture.png)

Therefore, DPDK vhost library has to transfer data between shared memory and DPDK mbuf. For this reason, vhost-user dequeue zero copy is introduced improve the performance from DPDK 16.11 release. Please note that this feature is only used for traffic from vhost-user (dequeue).

And I will investigate the detailed implementation in the next section.
# 2. Implementation Code
At first, there's a flag for vhost-user dequeue zero copy.
```
#define RTE_VHOST_USER_DEQUEUE_ZERO_COPY	(1ULL << 2)
```
And we should enable this flag while registering vhost driver.
```
int rte_vhost_driver_register(const char *path, uint64_t flags);
```
Let's take a close look at below function.
```
uint16_t rte_vhost_dequeue_burst(int vid, uint16_t queue_id,
	struct rte_mempool *mbuf_pool, struct rte_mbuf **pkts, uint16_t count);
```
This function gets guest buffers from the virtio device TX virtqueue, construct host mbufs, copies guest buffer content to host mbufs and store them in pkts to be processed.

This function supports to use zero copy operation to replace memory copy operation, and it also monitors used mbuf and notify guest that vring is updated.
## 2.1. Zero Copy
rte_vhost_dequeue_burst() uses copy_desc_to_mbuf() to perform zero copy operation or memory copy operation.
```
		err = copy_desc_to_mbuf(dev, vq, desc, sz, pkts[i], idx,
					mbuf_pool);
```
In copy_desc_to_mbuf(), if dequeue_zero_copy is enabled, descriptor address would be used as mbuf address, that's so-called zero copy operation. Otherwise, rte_memcpy() is used for memory copy operation.
```
		if (unlikely(dev->dequeue_zero_copy && (hpa = gpa_to_hpa(dev,
					desc_gaddr + desc_offset, cpy_len)))) {
			cur->data_len = cpy_len;
			cur->data_off = 0;
			cur->buf_addr = (void *)(uintptr_t)(desc_addr
				+ desc_offset);
			cur->buf_iova = hpa;

			/*
			 * In zero copy mode, one mbuf can only reference data
			 * for one or partial of one desc buff.
			 */
			mbuf_avail = cpy_len;
		} else {
			if (likely(cpy_len > MAX_BATCH_LEN ||
				   copy_nb >= vq->size ||
				   (hdr && cur == m) ||
				   desc->len != desc_chunck_len)) {
				rte_memcpy(rte_pktmbuf_mtod_offset(cur, void *,
								   mbuf_offset),
					   (void *)((uintptr_t)(desc_addr +
								desc_offset)),
					   cpy_len);
			} else {
				batch_copy[copy_nb].dst =
					rte_pktmbuf_mtod_offset(cur, void *,
								mbuf_offset);
				batch_copy[copy_nb].src =
					(void *)((uintptr_t)(desc_addr +
							     desc_offset));
				batch_copy[copy_nb].len = cpy_len;
				copy_nb++;
			}
		}
```
## 2.2. Monitoring Mbuf
At first, use a linked list to store used mbuf, and update refcnt to avoid wrongly releasing mbuf.
```
		if (unlikely(dev->dequeue_zero_copy)) {
			struct zcopy_mbuf *zmbuf;

			zmbuf = get_zmbuf(vq);
			if (!zmbuf) {
				rte_pktmbuf_free(pkts[i]);
				free_ind_table(idesc);
				break;
			}
			zmbuf->mbuf = pkts[i];
			zmbuf->desc_idx = desc_indexes[i];

			/*
			 * Pin lock the mbuf; we will check later to see
			 * whether the mbuf is freed (when we are the last
			 * user) or not. If that's the case, we then could
			 * update the used ring safely.
			 */
			rte_mbuf_refcnt_update(pkts[i], 1);

			vq->nr_zmbuf += 1;
			TAILQ_INSERT_TAIL(&vq->zmbuf_list, zmbuf, next);
		}
```
And then, ...
