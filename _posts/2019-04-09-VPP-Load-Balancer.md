---
layout: post
title:  "VPP Load Balancer module investigation"
date:   2019-04-09 12:00:00
categories: VPP
tags: VPP LB
excerpt: Analyze VPP Load Balancer module
mathjax: true
---

# 1. api
## 1.1. Introduction
This is not a node.

## 1.2. Interface
Initial function is lb_api_init().

And support below API message: 
```
/* List of message types that this plugin understands */
#define foreach_lb_plugin_api_msg            \
_(LB_CONF, lb_conf)                          \
_(LB_ADD_DEL_VIP, lb_add_del_vip)            \
_(LB_ADD_DEL_AS, lb_add_del_as)              \
_(LB_FLUSH_VIP, lb_flush_vip)
```

# 2. cli
## 2.1. Introduction
This is not a node.

## 2.2. Command
```
lb vip <prefix> 
[protocol (tcp|udp) port <n>] 
[encap (gre6|gre4|l3dsr|nat4|nat6)] 
[dscp <n>] 
[type (nodeport|clusterip) target_port <n>] 
[new_len <n>] [del]
```


