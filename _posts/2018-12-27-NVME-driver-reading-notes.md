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


