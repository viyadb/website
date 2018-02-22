Clustering
===========

Even though it's possible to run ViyaDB on a single node, it's usually required
to run several database instances in order to be able to:

 * Scale in case aggregated dataset doesn't fit into a single machine's RAM.
 * Scale out concurrent queries by spreading them between different workers.
 * Failover on data center failure.

This page describes how ViyaDB works when running in a cluster mode.

## Architecture

ViyaDB implements shared nothing architecture. That means that the data is partitioned
and spread between independent workers.

![Clustering](img/clustering1.png)

## Components

### Consul

Consul is used as a coordination service as well as for storing cluster configuration.

There were three different options for coordination service at the beginning: [Zookeeper](https://zookeeper.apache.org/),
[Consul](https://www.consul.io/) and [etcd](https://coreos.com/etcd/), but the choice was made in favor of Consul for its:

 * Installation and configuration simplicity.
 * Great Web interface.
 * Out of the box service registration and lookup support.

### Worker

Worker is a ViyaDB process that keeps a partition of data in RAM. Every such process is
attached to it's own CPU core (using CPU affinity) for achieving better cache locality.

Every worker listens on its own HTTP port, where it provides a service for loading and
querying the data it keeps.

### Controller

Controller is the main process that launches ViyaDB workers on a single node, and takes
care of keeping them alive. In addition, this process is responsible for coordination
between database nodes using Consul. One of cluster controllers becomes a leader.

Controller can act as query aggregator, which means that it can be queried directly, and
a query will be sent to workers containing required data partitions, then the result
will be combined and returned back to a user.

Communication with workers is performed using HTTP API.

## Real-Time Data Processing

To see the complete real-time data processing architecture based on ViyaDB cluster, please
visit [next](realtime.md) section.

