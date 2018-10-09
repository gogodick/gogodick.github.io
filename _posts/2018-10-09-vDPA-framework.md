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
