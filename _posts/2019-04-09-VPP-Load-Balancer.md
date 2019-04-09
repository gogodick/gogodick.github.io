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

And support below API messages: 
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
```
lb as <vip-prefix> [protocol (tcp|udp) port <n>]
 [<address> [<address> [...]]] [del] [flush]
```
```
lb conf [ip4-src-address <addr>] [ip6-src-address <addr>] [buckets <n>] [timeout <s>]
```
```
show lb
```
```
show lb vips [verbose]
```
```
lb set interface nat4 in <intfc> [del]
```
```
lb set interface nat6 in <intfc> [del]
```
```
lb flush vip <prefix> 
[protocol (tcp|udp) port <n>]
```
```
test lb flowtable flush
```

# 3. lb_test
## 3.1. Introduction
This is not a node.

# 4. lb
## 4.1. Introduction
This is not a node, and it registers load balancer plugin.

## 4.2. Interface
Initial function is lb_init()


