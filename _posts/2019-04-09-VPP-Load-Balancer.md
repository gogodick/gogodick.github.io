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
This module is not a node.

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
This module is not a node.

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
This module is not a node, and it registers load balancer plugin.

## 4.2. Interface
Initial function is lb_init()

# 5. node

## 5.1. Introduction
There are 16 nodes:
* lb6_gre6_node
* lb6_gre4_node
* lb4_gre6_node
* lb4_gre4_node
* lb6_gre6_port_node
* lb6_gre4_port_node
* lb4_gre6_port_node
* lb4_gre4_port_node
* lb4_l3dsr_port_node
* lb4_l3dsr_node
* lb6_nat6_port_node
* lb4_nat4_port_node
* lb4_nodeport_node
* lb6_nodeport_node
* lb_nat4_in2out_node
* lb_nat6_in2out_node

## 5.2. Detail
### 5.2.1. lb6_gre6_node
This node is using lb_node_fn(), is_input_v4 is 0, encap_type is LB_ENCAP_TYPE_GRE6, per_port_vip is 0.

* Call lb_node_get_hash() to compute hash value.
* If input is IPv4, get IPv4 header and header length.
* If input is IPv6, get IPv6 header and header length.
* Call lb_hash_get() to search load balancer hash table.
* If found an existing entry, do nothing.
* If there is an available slot for a new flow.
   * Update reference counter.
   * Call lb_hash_put() to add hash entry.
* If could not store new entry in the table, use conflict entry.
* If encap_type is LB_ENCAP_TYPE_GRE4.
   * Source address is load balancer source address.
   * Destination address is application server address.
   * Update some other field.
   * IP protocol is IP_PROTOCOL_GRE.
   * Update IP checksum.
   * protocol is 0x0800.
* If encap_type is LB_ENCAP_TYPE_GRE6.
   * Source address is load balancer source address.
   * Destination address is application server address.
   * Update some other field.
   * IP protocol is IP_PROTOCOL_GRE.
   * protocol is 0x86DD.
* If encap_type is LB_ENCAP_TYPE_L3DSR.
   * Replace destination address with application server address.
   * Move dscp to tos.
   * Update IP checksum.
   * Recomputing L4 checksum after dst-IP modifying.
* encap_type is LB_ENCAP_TYPE_NAT4.
   * If is_input_v4 is 1.
      * Replace destination address with application server address.
      * Update IP checksum.
      * If IP protocol is UDP.
         * Replace destination port with target port.
         * Update UDP checksum.
* encap_type is LB_ENCAP_TYPE_NAT6.
   * If is_input_v4 is 0.
      * Replace destination address with application server address.
      * If IP protocol is UDP.
         * Replace destination port with target port.
         * Update UDP checksum.

### 5.2.2. lb6_gre4_node
This node is using lb_node_fn(), is_input_v4 is 0, encap_type is LB_ENCAP_TYPE_GRE4, per_port_vip is 0.

### 5.2.3. lb4_gre6_node
This node is using lb_node_fn(), is_input_v4 is 1, encap_type is LB_ENCAP_TYPE_GRE6, per_port_vip is 0.

### 5.2.4. lb4_gre4_node
This node is using lb_node_fn(), is_input_v4 is 1, encap_type is LB_ENCAP_TYPE_GRE4, per_port_vip is 0.

### 5.2.5. lb6_gre6_port_node
This node is using lb_node_fn(), is_input_v4 is 0, encap_type is LB_ENCAP_TYPE_GRE6, per_port_vip is 1.

### 5.2.6. lb6_gre4_port_node
This node is using lb_node_fn(), is_input_v4 is 0, encap_type is LB_ENCAP_TYPE_GRE4, per_port_vip is 1.

### 5.2.7. lb4_gre6_port_node
This node is using lb_node_fn(), is_input_v4 is 1, encap_type is LB_ENCAP_TYPE_GRE6, per_port_vip is 1.

### 5.2.8. lb4_gre4_port_node
This node is using lb_node_fn(), is_input_v4 is 1, encap_type is LB_ENCAP_TYPE_GRE4, per_port_vip is 1.

### 5.2.9. lb4_l3dsr_port_node
This node is using lb_node_fn(), is_input_v4 is 1, encap_type is LB_ENCAP_TYPE_L3DSR, per_port_vip is 1.

### 5.2.10. lb4_l3dsr_node
This node is using lb_node_fn(), is_input_v4 is 1, encap_type is LB_ENCAP_TYPE_L3DSR, per_port_vip is 0.

### 5.2.11. lb6_nat6_port_node
This node is using lb_node_fn(), is_input_v4 is 0, encap_type is LB_ENCAP_TYPE_NAT6, per_port_vip is 1.

### 5.2.12. lb4_nat4_port_node
This node is using lb_node_fn(), is_input_v4 is 1, encap_type is LB_ENCAP_TYPE_NAT4, per_port_vip is 1.

### 5.2.13. lb4_nodeport_node
This node is using lb_nodeport_node_fn(), is_input_v4 is 1.

### 5.2.14. lb6_nodeport_node
This node is using lb_nodeport_node_fn(), is_input_v4 is 0.

### 5.2.15. lb_nat4_in2out_node
This node is using lb_nat_in2out_node_fn(), is_nat4 is 1.

### 5.2.16. lb_nat6_in2out_node
This node is using lb_nat_in2out_node_fn(), is_nat4 is 0.

# 6. util

## 6.1. Introduction
This module is not a node.
