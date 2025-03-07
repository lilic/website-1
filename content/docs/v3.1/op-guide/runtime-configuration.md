---
title: Runtime reconfiguration
---

etcd comes with support for incremental runtime reconfiguration, which allows users to update the membership of the cluster at run time.

Reconfiguration requests can only be processed when the majority of the cluster members are functioning. It is **highly recommended** to always have a cluster size greater than two in production. It is unsafe to remove a member from a two member cluster. The majority of a two member cluster is also two. If there is a failure during the removal process, the cluster might not able to make progress and need to [restart from majority failure][majority failure].

To better understand the design behind runtime reconfiguration, we suggest reading [the runtime reconfiguration document][runtime-reconf].

## Reconfiguration use cases

Let's walk through some common reasons for reconfiguring a cluster. Most of these just involve combinations of adding or removing a member, which are explained below under [Cluster Reconfiguration Operations][cluster-reconf].

### Cycle or upgrade multiple machines

If multiple cluster members need to move due to planned maintenance (hardware upgrades, network downtime, etc.), it is recommended to modify members one at a time.

It is safe to remove the leader, however there is a brief period of downtime while the election process takes place. If the cluster holds more than 50MB, it is recommended to [migrate the member's data directory][member migration].

### Change the cluster size

Increasing the cluster size can enhance [failure tolerance][fault tolerance table] and provide better read performance. Since clients can read from any member, increasing the number of members increases the overall read throughput.

Decreasing the cluster size can improve the write performance of a cluster, with a trade-off of decreased resilience. Writes into the cluster are replicated to a majority of members of the cluster before considered committed. Decreasing the cluster size lowers the majority, and each write is committed more quickly.

### Replace a failed machine

If a machine fails due to hardware failure, data directory corruption, or some other fatal situation, it should be replaced as soon as possible. Machines that have failed but haven't been removed adversely affect the quorum and reduce the tolerance for an additional failure.

To replace the machine, follow the instructions for [removing the member][remove member] from the cluster, and then [add a new member][add member] in its place. If the cluster holds more than 50MB, it is recommended to [migrate the failed member's data directory][member migration] if it is still accessible.

### Restart cluster from majority failure

If the majority of the cluster is lost or all of the nodes have changed IP addresses, then manual action is necessary to recover safely.
The basic steps in the recovery process include [creating a new cluster using the old data][disaster recovery], forcing a single member to act as the leader, and finally using runtime configuration to [add new members][add member] to this new cluster one at a time.

## Cluster reconfiguration operations

Now that we have the use cases in mind, let us lay out the operations involved in each.

Before making any change, the simple majority (quorum) of etcd members must be available.
This is essentially the same requirement as for any other write to etcd.

All changes to the cluster are done one at a time:

* To update a single member peerURLs, make an update operation
* To replace a single member, make an add then a remove operation
* To increase from 3 to 5 members, make two add operations
* To decrease from 5 to 3, make two remove operations

All of these examples will use the `etcdctl` command line tool that ships with etcd.
To change membership without `etcdctl`, use the [v2 HTTP members API][member-api] or the [v3 gRPC members API][member-api-grpc].

### Update a member

#### Update advertise client URLs

To update the advertise client URLs of a member, simply restart
that member with updated client urls flag (`--advertise-client-urls`) or environment variable
(`ETCD_ADVERTISE_CLIENT_URLS`). The restarted member will self publish the updated URLs.
A wrongly updated client URL will not affect the health of the etcd cluster.

#### Update advertise peer URLs

To update the advertise peer URLs of a member, first update 
it explicitly via member command and then restart the member. The additional action is required
since updating peer URLs changes the cluster wide configuration and can affect the health of the etcd cluster. 

To update the peer URLs, first, we need to find the target member's ID. To list all members with `etcdctl`:

```sh
$ etcdctl member list
6e3bd23ae5f1eae0: name=node2 peerURLs=http://localhost:23802 clientURLs=http://127.0.0.1:23792
924e2e83e93f2560: name=node3 peerURLs=http://localhost:23803 clientURLs=http://127.0.0.1:23793
a8266ecf031671f3: name=node1 peerURLs=http://localhost:23801 clientURLs=http://127.0.0.1:23791
```

In this example let's `update` a8266ecf031671f3 member ID and change its peerURLs value to http://10.0.1.10:2380

```sh
$ etcdctl member update a8266ecf031671f3 http://10.0.1.10:2380
Updated member with ID a8266ecf031671f3 in cluster
```

### Remove a member

Let us say the member ID we want to remove is a8266ecf031671f3.
We then use the `remove` command to perform the removal:

```sh
$ etcdctl member remove a8266ecf031671f3
Removed member a8266ecf031671f3 from cluster
```

The target member will stop itself at this point and print out the removal in the log:

```
etcd: this member has been permanently removed from the cluster. Exiting.
```

It is safe to remove the leader, however the cluster will be inactive while a new leader is elected. This duration is normally the period of election timeout plus the voting process.

### Add a new member

Adding a member is a two step process:

 * Add the new member to the cluster via the [HTTP members API][member-api], the [gRPC members API][member-api-grpc], or the `etcdctl member add` command.
 * Start the new member with the new cluster configuration, including a list of the updated members (existing members + the new member).

Using `etcdctl` let's add the new member to the cluster by specifying its [name][conf-name] and [advertised peer URLs][conf-adv-peer]:

```sh
$ etcdctl member add infra3 http://10.0.1.13:2380
added member 9bf1b35fc7761a23 to cluster

ETCD_NAME="infra3"
ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra3=http://10.0.1.13:2380"
ETCD_INITIAL_CLUSTER_STATE=existing
```

`etcdctl` has informed the cluster about the new member and printed out the environment variables needed to successfully start it.
Now start the new etcd process with the relevant flags for the new member:

```sh
$ export ETCD_NAME="infra3"
$ export ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra3=http://10.0.1.13:2380"
$ export ETCD_INITIAL_CLUSTER_STATE=existing
$ etcd --listen-client-urls http://10.0.1.13:2379 --advertise-client-urls http://10.0.1.13:2379 --listen-peer-urls http://10.0.1.13:2380 --initial-advertise-peer-urls http://10.0.1.13:2380 --data-dir %data_dir%
```

The new member will run as a part of the cluster and immediately begin catching up with the rest of the cluster.

If adding multiple members the best practice is to configure a single member at a time and verify it starts correctly before adding more new members.
If adding a new member to a 1-node cluster, the cluster cannot make progress before the new member starts because it needs two members as majority to agree on the consensus. This behavior only happens between the time `etcdctl member add` informs the cluster about the new member and the new member successfully establishing a connection to the existing one.

#### Error cases when adding members

In the following case we have not included our new host in the list of enumerated nodes.
If this is a new cluster, the node must be added to the list of initial cluster members.

```sh
$ etcd --name infra3 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state existing
etcdserver: assign ids error: the member count is unequal
exit 1
```

In this case we give a different address (10.0.1.14:2380) to the one that we used to join the cluster (10.0.1.13:2380).

```sh
$ etcd --name infra4 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra4=http://10.0.1.14:2380 \
  --initial-cluster-state existing
etcdserver: assign ids error: unmatched member while checking PeerURLs
exit 1
```

When we start etcd using the data directory of a removed member, etcd will exit automatically if it connects to any active member in the cluster:

```sh
$ etcd
etcd: this member has been permanently removed from the cluster. Exiting.
exit 1
```

### Strict reconfiguration check mode (`-strict-reconfig-check`)

As described in the above, the best practice of adding new members is to configure a single member at a time and verify it starts correctly before adding more new members. This step by step approach is very important because if newly added members is not configured correctly (for example the peer URLs are incorrect), the cluster can lose quorum. The quorum loss happens since the newly added member are counted in the quorum even if that member is not reachable from other existing members. Also quorum loss might happen if there is a connectivity issue or there are operational issues.

For avoiding this problem, etcd provides an option `-strict-reconfig-check`. If this option is passed to etcd, etcd rejects reconfiguration requests if the number of started members will be less than a quorum of the reconfigured cluster.

It is enabled by default.

[add member]: #add-a-new-member
[cluster-reconf]: #cluster-reconfiguration-operations
[conf-adv-peer]: configuration#--initial-advertise-peer-urls
[conf-name]: configuration#--name
[disaster recovery]: recovery
[majority failure]: #restart-cluster-from-majority-failure
[member-api-grpc]: ../dev-guide/api_reference_v3#service-cluster-etcdserveretcdserverpbrpcproto
[remove member]: #remove-a-member
[runtime-reconf]: runtime-reconf-design
[member-api]: /docs/v2.3/members_api
[member migration]: /docs/v2.3/admin_guide#member-migration
[fault tolerance table]: /docs/v2.3/admin_guide#fault-tolerance-table
