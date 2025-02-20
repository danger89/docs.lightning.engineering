# Clustering

Normally LND nodes use the embedded bbolt database to store all important states. This method of running has been proven to work well in a variety of environments, from mobile clients to large nodes serving hundreds of channels. With scale however it is desirable to be able to replicate LND's state to quickly and reliably move nodes, do updates and be more resilient to datacenter failures.

It is now possible to store all essential state in a replicated etcd DB and to run multiple LND nodes on different machines where only one of them \(the leader\) is able to read and mutate the database. In such setup if the leader node fails or decomissioned, a follower node will be elected as the new leader and will quickly come online to minimize downtime.

The leader election feature currently relies on etcd to work both for the election itself and for the replicated data store.

## Building LND with leader election support

To create a dev build of LND with leader election support use the following command:

```text
⛰  make tags="kvdb_etcd"
```

## Running a local etcd instance for testing

To start your local etcd instance for testing run:

```text
⛰  ./etcd \
    --auto-tls \
    --advertise-client-urls=https://127.0.0.1:2379 \
    --listen-client-urls=https://0.0.0.0:2379 \
    --max-txn-ops=16384 \
    --max-request-bytes=104857600
```

The large `max-txn-ops` and `max-request-bytes` values are currently recommended but may not be required in the future.

## Configuring LND to run on etcd and participate in leader election

To run LND with etcd, additional configuration is needed, specified either through command line flags or in `lnd.conf`.

Sample command line:

```text
⛰  ./lnd-debug \
    --db.backend=etcd \
    --db.etcd.host=127.0.0.1:2379 \
    --db.etcd.certfile=/home/user/etcd/bin/default.etcd/fixtures/client/cert.pem \
    --db.etcd.keyfile=/home/user/etcd/bin/default.etcd/fixtures/client/key.pem \
    --db.etcd.insecure_skip_verify \
    --cluster.enable-leader-election \
    --cluster.leader-elector=etcd \
    --cluster.etcd-election-prefix=cluster-leader \
    --cluster.id=lnd-1
```

The `cluster.etcd-election-prefix` option sets the election's etcd key prefix. The `cluster.id` is used to identify the individual nodes in the cluster and should be set to a different value for each node.

Optionally users can specify `db.etcd.user` and `db.etcd.pass` for db user authentication. If the database is shared, it is possible to separate our data from other users by setting `db.etcd.namespace` to an \(already existing\) etcd namespace. In order to test without TLS, we can set `db.etcd.disabletls` flag to `true`.

Once the node is up and running we can start more nodes with the same command line.

## Identifying the leader node

The above setup is useful for testing but is not viable when running in a production environment. For users relying on containers and orchestration services, it is essential to know which node is the leader to be able to automatically route network traffic to the right instance. For example in Kubernetes, the load balancer will route traffic to all "ready" nodes. This readiness may be monitored by a readiness probe.

For readiness probing we can simply use LND's state RPC service where a special state `WAITING_TO_START` indicates that the node is waiting to become the leader and is not started yet. To test this we can simply curl the REST endpoint of the state RPC:

```text
readinessProbe:
    exec:
      command: [
        "/bin/sh",
        "-c",
        "set -e; set -o pipefail; curl -s -k -o - https://localhost:8080/v1/state | jq .'State' | grep -E 'NON_EXISTING|LOCKED|UNLOCKED|RPC_ACTIVE'",
      ]
    periodSeconds: 1
```

## Replication of non-critical data

All critical data is written to the replicated database, including LND's wallet data which contains the key material and node identity. Some less critical data however is currently not written to that same database for performance reasons and is instead still kept in local `bbolt` files.

For example the graph data is kept locally to improve path finding. Other examples are the macaroon database or watchtower client database. To make sure a node can become active and take over quickly if the leader fails, it is therefore still recommended to have the LND data directory on a shared volume that all active and passive nodes can access. Otherwise the node that is taking over might first need to sync its graph.

As we evolve our cluster support we'll provide more solutions to make replication and clustering even more seamless.

