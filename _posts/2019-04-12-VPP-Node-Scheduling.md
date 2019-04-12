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
VPP support 2 kinds of thread: main thread and worker thread.

# 1.1. Main thread
Calling sequence is thread0()->vlib_main()->vlib_main_loop()->vlib_main_or_worker_loop().

# 1.2. Worker thread

# 2. Node
