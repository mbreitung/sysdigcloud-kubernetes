# Datastores as Kubernetes pod

Sysdig Cloud datastores can be deployed as Kubernetes pods. Each pod can be configured to use a
local volume (emptyDir) that is only persisted for the lifetime of the pod, or a persistent volume.

## MySQL Deployment or StatefulSet

### Deployment (without HA, simple and recommended)

To create a MySQL deployment, the provided manifest under
`manifests/mysql/mysql-deployment.yaml` can be used. By default, it will
use a local non-persistent volume (emptyDir), but the manifest contains
commented snippets that can be uncommented when using persistent volumes
such as AWS EBS or GCE Disks (just replace the volume id from the cloud
provider in the snippet):

```
kubectl -n sysdigcloud create -f manifests/mysql/mysql-deployment.yaml
```

If testing at high scale, note the increased `max_connections` setting in
`manifests/mysql.yaml`.

This solution creates a single replica instance of MySQL. The
implementation is very robust and based on a greatly stable MySQL release
(5.6), but it doesn't have HA capabilities, meaning that if the instance
running MySQL fails, the application will face downtime until Kubernetes
reschedules the pod on another healthy node and reattaches to it the
persistent storage. This downtime can tipically last from a few seconds to
a few minutes. If more HA is needed, a solution based on StatefulSets can
be used.

### StatefulSet (with HA, advanced)

This solution is based on StatefulSet and a more recent version of MySQL
(8.0.11) with significant capabilities in terms of HA.

It can be deployed with:

```
kubectl -n sysdigcloud create -f manifests/mysql/mysql-cluster-statefulset.yaml
kubectl -n sysdigcloud create -f manifests/mysql/mysql-router.yaml
```

This creates a MySQL InnoDB Cluster of 3 members in single-primary mode,
with automatic failover and replication. In this solution, users can incur
into a loss of one MySQL instance at a time without causing any significant
downtime, as the failover towards a healthy instance should be instantaneous.
Once the failed instance recovers, it will automatically join as secondary
replica to the existing cluster.

In front of the cluster, a stateless deployment of 3 instances of MySQL
Router is instantiated, so that clients can always be correctly routed
towards the current MySQL master.

Each MySQL instance is deployed with a sidecar agent that should take care
of the basic cluster membership administrative operations. Unless major
outages or particularly bad failures happen, it shouldn't be necessary for
the user to issue any manual intervention. However, with MySQL not being a
distributed system by nature, some circumstances can cause the cluster to
misbehave, and manual intervention might be needed. The following resources
can be used:

- https://dev.mysql.com/doc/refman/8.0/en/mysql-innodb-cluster-working-with-cluster.html

- https://dev.mysql.com/doc/refman/8.0/en/mysql-innodb-cluster-production-deployment.html

If the user doesn't want to incur into the maintenance overhead of a
replicated MySQL, simply using a single deployment instance with
persistence volume will work as well, with the reduced HA as explained in
the section above.

## Redis Deployment

### Deployment

```
kubectl -n sysdigcloud create -f manifests/redis/redis-deployment.yaml
```

## Cassandra Deployment OR Statefulset

### Statefulset

To create a Cassandra statefulset, the provided manifest under `manifests/cassandra/cassandra-statefulset.yaml`
can be used. By default, it will use a local non-persistent volume (standard dir).

```
kubectl -n sysdigcloud create -f manifests/cassandra/cassandra-service.yaml
kubectl -n sysdigcloud create -f manifests/cassandra/cassandra-statefulset.yaml
```

This creates a Cassandra cluster of size 3. To expand the Cassandra cluster, change the `replicas` to the
desired higher number.

### Deployment (deprecated)

Before deploying the deployment object, the proper Cassandra headless service must be created (the headless
service will be used for service discovery when deploying a multi-node Cassandra cluster):

```
kubectl -n sysdigcloud create -f manifests/cassandra/cassandra-service.yaml
```

To create a Cassandra deployment, the provided manifest under `manifests/cassandra/cassandra-deployment.yaml`
can be used. By default, it will use a local non-persistent volume (emptyDir), but the manifest
contains commented snippets that can be uncommented when using persistent volumes such as AWS EBS or
GCE Disks (just replace the volume id from the cloud provider in the snippet):

```
kubectl -n sysdigcloud create -f manifests/cassandra/cassandra-deployment.yaml --namespace sysdigcloud
```

This creates a Cassandra cluster of size 1. To expand the Cassandra cluster, a new deployment must
be created for each additional Cassandra node in the cluster. You cannot simply scale the replicas of
the existing deployment because each Cassandra pod must get a different persistent volume, so in
that sense Cassandra pods are "pets" with unique identities and not "cattle".

In order for the new Cassandra deployment to automatically join the cluster, some conventions must
be followed. In particular, the Cassandra node number (1, 2, 3, ...) must be properly put in the
manifest `manifests/cassandra/cassandra-deployment.yaml` under the entries marked as `# Cassandra node number`.

For example, to scale a Cassandra cluster from 2 to 3 nodes, the manifest can be edited as such:

```
...
metadata:
  name: sysdigcloud-cassandra-3 # Cassandra node number
...
      labels:
        instance: "3" # Cassandra node number
...
```

And then the deployment can be created as usual:

```
kubectl -n sysdigcloud create -f manifests/cassandra/cassandra-deployment.yaml
```

After each scaling activity, the status of the cluster can be checked by executing
`nodetool status` in one of the Cassandra pods. All the Cassandra nodes should be listed
as `UN` in order for the cluster to be fully up and running. Immediately after the scaling
activity, the new pod will be in joining phase:

```
$ kubectl -n sysdigcloud exec -it sysdigcloud-cassandra-1-2987866586-f5kgo -- nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens  Owns (effective)  Host ID                               Rack
UN  10.52.2.4  1.88 MB    256     54.4%             99121365-4543-4e50-ae6f-a9a9cb720b7c  rack1
UJ  10.52.0.4  14.43 KB   256     ?                 4b084d81-21f1-45b6-add9-8fbea7392978  rack1
UN  10.52.1.7  917.91 KB  256     45.6%             9a7437e9-890f-477a-99be-3d8042ddd9d5  rack1
```

After the bootstrapping process terminates, the new pod will terminate the joining phase and the
cluster will be fully operational:

```
$ kubectl -n sysdigcloud exec -it sysdigcloud-cassandra-1-2987866586-f5kgo -- nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens  Owns (effective)  Host ID                               Rack
UN  10.52.2.4  1.88 MB    256     34.1%             99121365-4543-4e50-ae6f-a9a9cb720b7c  rack1
UN  10.52.0.4  14.43 KB   256     34.0%             4b084d81-21f1-45b6-add9-8fbea7392978  rack1
UN  10.52.1.7  917.91 KB  256     31.9%             9a7437e9-890f-477a-99be-3d8042ddd9d5  rack1
```

It is important for the number of deployments to be increased by no more than 1 at every scaling
activity, since Cassandra will refuse the joining of a new node in the Cluster if one joining
process is already in progress.

Maintaining a multi-node production Cassandra cluster requires some simple but mandatory housekeeping
procedures, best described in the official documentation.

## Elasticsearch

### Statefulset

To create a Elasticsearch statefulset, the provided manifest under
`manifests/elasticsearch/elasticsearch-statefulset.yaml`
can be used. By default, it will use a local non-persistent volume (standard dir).

```
kubectl -n sysdigcloud create -f manifests/elasticsearch/elasticsearch-service.yaml
kubectl -n sysdigcloud create -f manifests/elasticsearch/elasticsearch-statefulset.yaml
```

This creates a Elasticsearch cluster of size 3. To expand the Elasticsearch cluster, change the `replicas` to the
desired higher number.

### Deployment (deprecated)

Before deploying the deployment object, the proper Elasticsearch headless service must be created
(the headless service will be used for service discovery when deploying a multi-node Elasticsearch cluster):

```
kubectl -n sysdigcloud create -f manifests/elasticsearch/elasticsearch-service.yaml
```
To create an Elasticsearch deployment, the provided manifest under `manifests/elasticsearch-deployment.yaml`
can be used. By default, it will use a local non-persistent volume (emptyDir), but the manifest contains
commented snippets that can be uncommented when using persistent volumes such as AWS EBS or GCE Disks
(just replace the volume id from the cloud provider in the snippet):

```
kubectl -n sysdigcloud create -f manifests/elasticsearch/elasticsearch-deployment.yaml
```
This creates an Elasticsearch cluster of size 1. To expand the Elasticsearch cluster, a new deployment
must be created for each additional Elasticsearch node in the cluster. You can't just scale the replicas
of the existing deployment because each Elasticsearch pod must get a different persistent volume, so in
that sense Elasticsearch pods are "pets" with unique identities and not "cattle".

In order for the new Elasticsearch deployment to automatically join the cluster, some conventions must
be followed. In particular, the Elasticsearch node number (1, 2, 3, ...) must be properly put in the
manifest `manifests/elasticsearch/elasticsearch-deployment.yaml` under the entries marked as `# Elasticsearch node number`.

For example, to scale the Elasticsearch cluster from 2 to 3 nodes, the manifest can be edited as such:

```
...
metadata:
  name: sysdigcloud-elasticsearch-3 # Elasticsearch node number
...
      labels:
        instance: "3" # Elasticsearch node number
...
```

And then the deployment can be created as usual:

```
kubectl -n sysdigcloud create -f manifests/elasticsearch-deployment.yaml
```
After each scaling activity, the status of the cluster can be checked by executing
`curl -sS http://127.0.0.1:9200/_cluster/health?pretty=true` in one of the Elasticsearch pods.

```
$ kubectl -n sysdigcloud exec -it sysdigcloud-elasticsearch-1-2660816362-tfht5 -- curl -sS http://127.0.0.1:9200/_cluster/health?pretty=true
{
  "cluster_name" : "sysdigcloud",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 2,
  "number_of_data_nodes" : 2,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```
