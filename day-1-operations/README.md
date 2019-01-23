---
description: 第一天要学习的内容，关于如何部署一个数据中心。
---

# 进阶教程\(一\)

This learning path is designed to help you deploy your first datacenter. If you are responsible for setting up and maintaining a healthy datacenter, this path will help you do so successfully. We will cover the following topics:

* Infrastructure recommendations
* Setting up a datacenter
* Backing up the state of the datacenter
* Securing the datacenter
* Ensuring the datacenter is healthy

Below you will find all of the guides that make up this learning path. If you want to skip ahead to any guide, feel free to use them as needed. Each guide has a description along with its objective to help you decide. However, if you are deploying your first production ready datacenter, we recommend completing these guides in order.

### Reference Architecture <a id="reference-architecture"></a>

By the end of this guide you will be ready to create a architecture diagram for your environment. You will be able to identify which ports should be open, select hardware sizes that meet your needs, and understand how to implement datacenter design best practices.

[Reference Architecture](https://learn.hashicorp.com/consul/advanced/day-1-operations/reference-architecture)

### Deployment Guide <a id="deployment-guide"></a>

By the end of the guide you will install and configure a single Consul cluster. You will use the examples to create your own custom configuration files for both servers and clients. The custom configuration files will help you join agents, optimize Raft performance, enable the collection of metrics, and configure the web UI. Finally, the guide will detail how to configure Systemd.

[Deployment Guide](https://learn.hashicorp.com/consul/advanced/day-1-operations/deployment-guide)

### Datacenter Backups <a id="datacenter-backups"></a>

By the end of this guide, you will have a backup process outlined. You will also be able to list the server data that is saved. Finally, you will understand the process for restoring from a backup.

[Datacenter Backups](https://learn.hashicorp.com/consul/advanced/day-1-operations/backup)

### Bootstrapping ACL System <a id="bootstrapping-acl-system"></a>

By the end of this guide you will have ACLs configured on the Consul agents, servers and clients. For each step, you will be able to recognize if the process is not properly executed. Optionally, you can also configure the _anonyomous token_ and token for the UI.

[Bootstrapping ACLs](https://learn.hashicorp.com/consul/advanced/day-1-operations/acl-guide)

### Creating Agent Certificates <a id="creating-agent-certificates"></a>

By the end of the this guide you will know how to generate certificates for your cluster. This guide will cover how to create a Certificate Authority\(CA\), and how to generate server certificates and client certificates.

[Creating Agent Certificates](https://learn.hashicorp.com/consul/advanced/day-1-operations/certificates)

### Gossip and RPC Encryption <a id="gossip-and-rpc-encryption"></a>

By the end of this guide you will be able to configure gossip and RPC encryption on a your cluster. Encrypting both incoming and outgoing communication is crucial for securing the cluster.

[Gossip and RPC Encryption](https://learn.hashicorp.com/consul/advanced/day-1-operations/agent-encryption)

### Monitor Cluster Health <a id="monitor-cluster-health"></a>

By the end of the monitoring guide, you will be able to collect agent metrics. You will be able to identify the various methods for collecting metrics and configure them for your use cases; quick collection or to use with monitoring software. You will also be able to interpret the key metrics for a healthy cluster.

[Monitor Cluster Health](https://learn.hashicorp.com/consul/advanced/day-1-operations/monitoring)

### Get Started <a id="get-started"></a>

Now that we have reviewed the guides that make up the Day 1 Operations learning path, get started by either hitting the next button at the bottom of the page or select the guide that you are interested in. If you encounter any technical difficulties while working through the guides or have any feedback please send an email to the [Consul mailing list](https://groups.google.com/forum/#!forum/consul-tool).

