---
layout: post
title:  "Investigating node scheduling for VPP"
date:   2019-04-12 12:00:00
categories: VPP
tags: VPP node thread
excerpt: Analyze the implementation of VPP node scheduling mechanism
mathjax: true
---
# 1. Thread
VPP supports 2 kinds of thread: main thread and worker thread.

## 1.1. Main thread
Calling sequence is thread0()->vlib_main()->vlib_main_loop()->vlib_main_or_worker_loop().

## 1.2. Worker thread
Calling sequence is vlib_worker_thread_fn()->vlib_worker_loop()->vlib_main_or_worker_loop().

## 1.3. Thread synchronization
Please refer to below functions:
* vlib_worker_thread_barrier_check()
* vlib_worker_thread_barrier_sync()
* vlib_worker_thread_barrier_release()

# 2. Node
VPP supports 4 kinds of node:
* VLIB_NODE_TYPE_INTERNAL
* VLIB_NODE_TYPE_INPUT
* VLIB_NODE_TYPE_PRE_INPUT
* VLIB_NODE_TYPE_PROCESS

Please refer to vlib_main_or_worker_loop() for details.

## 2.1. VLIB_NODE_TYPE_INTERNAL
An internal node on the call graph (could be output).

These nodes are invoked in main thread and worker thread.
```
cpu_time_now = dispatch_pending_node (vm, i, cpu_time_now);
```

## 2.2. VLIB_NODE_TYPE_INPUT
Nodes which input data into the processing graph. Input nodes are called for each iteration of main loop.

These nodes are invoked in main thread and worker thread.
```
	cpu_time_now = dispatch_node (vm, n,
				      VLIB_NODE_TYPE_INPUT,
				      VLIB_NODE_STATE_POLLING,
				      /* frame */ 0,
				      cpu_time_now);
```
## 2.3. VLIB_NODE_TYPE_PRE_INPUT
Nodes to be called before all input nodes. Used, for example, to clean out driver TX rings before processing input.

These nodes are invoked in main thread and worker thread.
```
	cpu_time_now = dispatch_node (vm, n,
				      VLIB_NODE_TYPE_PRE_INPUT,
				      VLIB_NODE_STATE_POLLING,
				      /* frame */ 0,
				      cpu_time_now);
```

## 2.4. VLIB_NODE_TYPE_PROCESS
"Process" nodes which can be suspended and later resumed. For example, unix_cli_process().

These nodes are invoked in main thread.
```
	cpu_time_now = dispatch_process (vm, nm->processes[i], /* frame */ 0,
					 cpu_time_now);
```
```
	dispatch_suspended_process (vm, di, cpu_time_now);
```

