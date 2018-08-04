---
title: 初学Zabbix基本原理
date: 2017-10-21 15:21:45
tags:
- Zabbix
categories:
- DevOps
---

对 Zabbix 的了解一直停留在听说过没用过的状态，以前通过 python-zabbix 调用过简单的 API，但是讲原理的话我真的说不上来什么。很开心新的工作对我提出了新的要求，我的 Mentor——Rick 也是个很好的老师，从基本原理和架构上给我介绍了 Zabbix 到底是什么。下面是我的学习整理。
<!--more-->

## Zabbix 中的几个重要概念

- host
    a networked device that you want to monitor, with IP/DNS.

- host group
    a logical grouping of hosts;
    it may contain hosts and templates.
    Hosts and templates within a host group are not in any way linked to each other.
    Host groups are used when assigning access rights to hosts for different user groups.

- item
    a particular piece of data that you want to receive off of a host, a metric of data.

- application
    a grouping of items in a logical group.

- trigger
    a logical expression that defines a problem threshold and is used to “evaluate” data received in items.
    When received data are above the threshold, triggers go from 'Ok' into a 'Problem' state. When received data are below the threshold, triggers stay in/return to an 'Ok' state.

- event
    a single occurrence of something that deserves attention such as a trigger changing state or a discovery/agent auto-registration taking place.

- action
    a predefined means of reacting to an event.
    An action consists of operations (e.g. sending a notification) and conditions (when the operation is carried out)

- template
    a set of entities (items, triggers, graphs, screens, applications, low-level discovery rules, web scenarios) ready to be applied to one or several hosts.
    The job of templates is to speed up the deployment of monitoring tasks on a host; also to make it easier to apply mass changes to monitoring tasks.
    Templates are linked directly to individual hosts.



## Zabbix 有两种模式

- 主动模式（对服务器开销较小，适合大规模监控环境）
- 被动模式（对服务器开销较大）

## Zabbix 的两种架构

- Agent -> Server (结构简单，适合规模较小，处于同一地域的环境)
- Agent -> Proxy -> Server (适合大规模，监控节点多，监控类型多，监控数据和网络连接开销大，跨地域的环境)

### Server -> Proxy -> Agent

![zabbix代理架构](zabbix代理架构.png)

1. 在 Host 上安装 Agent
2. Agent 的配置文件 zabbix_agentd.conf 中有 ServerActive，RefreshActiveChecks，BufferSend
3. Agent 按指定的 RefreshActiveChecks 去指定的 ServerActive 上请求需要采集的指标项列表，然后采集信息
4. Agent 按指定的 BufferSend 间隔向 Proxy 上报本周期采集到的信息
5. Proxy 按 zabbix_proxy.conf 中指定的间隔时间向 Server 上报本周期从 Agent 收到的信息

信息采集周期示例：
假设每 1s 采集一次 CPU 信息，
每 2s 采集一次 Mem 信息
zabbix_agentd.conf 中设定的 BufferSend 为 5s
zabbix_proxy.conf 中设定的 DataSenderFrequency 为 6s

如图所示：
![Zabbix上报数据图示](zabbix数据上报图示.png)

那么在第 10s 的时刻：
Agent 对 CPU 的信息采集了 10 次
Agent 对 Mem 的信息采集了 5 次
Agent 向 Proxy 上报了 2 次（包含 10 次 CPU 的信息和 5 次 Mem 的信息）
Proxy 向 Server 上报了一次（包含 5 次 CPU 的信息和 2 次 Mem 的信息）

## Zabbix 自动发现

在 Zabbix 中手动创建 host 不仅繁琐，易出错，可能也不够及时。
Zabbix 提供了自动发现的功能，通过创建 Discovery rules 来检查是否有新的 host。

Discovery rule 的配置：
- Name： 规则名称
- Discovery by proxy: 通过哪个 Proxy 去检测
- IP range：指定要检测的 IP 列表或网段
- Delay（in sec）：检测周期
- Checks：使用哪种方式检测，支持SSH，HTTP，HTTPS，FTP等多种协议和 Zabbix agent 方式，可以指定端口或端口范围
- Device uniqueness criteria：配置使用哪一项作为设备的唯一标识
- Enabled： 是否启用规则

对 Discovery rule 也可以配置相应的 Action，当满足某种条件是触发指定的动作。
如：发现新的 host 时，划分到指定的 hostgroup，连接到某一个 template 等。

