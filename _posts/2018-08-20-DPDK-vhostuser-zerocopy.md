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

Therefore, DPDK vhost library has to transfer data between shared memory and DPDK mbuf. For this reason, vhost-user dequeue zero copy is introduced from DPDK 16.11 release. And I will investigate the detailed implementation in the next section.

# 2. Implementation Code
