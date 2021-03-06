# Rolling Upgrades:  Neo4j in Kubernetes

This document expands on the Neo4j Operations Manual entry
[Upgrade a Causal Cluster](https://neo4j.com/docs/operations-manual/current/upgrade/causal-cluster/) with information about approaches on rolling upgrades in Kubernetes.

**It is strongly recommended that you read all of that documentation before using this method**.

Not all relevant concepts will be described in this document.  Familiarity with the page above will be assumed.

**It is recommended to perform a test upgrade on a production-like environment to get information on the duration of the downtime, if any, that may be necessary.**

## When is this needed?

* When you have a Neo4j Causal Cluster (standalone does not apply)
* When you need to upgrade to a new minor or patch version of Neo4j
* When you must maintain the cluster online with both read and write capabilities during the course of the upgrade process.

## What This Approach Doesn't Cover

Moving between major versions of Neo4j (for example, 3.5 and 4.0).  
This requires more planning, due to these factors:

* Substantial configuration changes needed on the pods between 3.5 and 4.0
* Major changes in product features which impact what clients can rely upon.
* Changes in the helm charts used, and their structure; if you're using an old
3.5 helm chart, the differences between what you're using and this repo may be substantial.
* The need for a store upgrade operation to change the format of Neo4j data on disk

Neo4j 4.0 is not backwards compatible with 3.5. Additional planning and offline upgrade is recommended.

If you are in the situation of migrating from Neo4j 3.5 -> 4.0, please consult
[the Neo4j Migration Guide](https://neo4j.com/docs/migration-guide/current/).

If this won't work for your constraints, please check "Alternatives to Rolling Upgrades" 
at the very bottom of this document.

## High-Level Approach

1. Take a backup
2. Scale the core statefulset up, to maintain high availability. 
3. Choose and apply your UpdateStrategy.
4. Patch the statefulset to apply the new Neo4j version
5. Monitor the process
6. (Optional/If Applicable) Apply the above process to the read replica StatefulSet as well.
7. Scale back down on success to the original size.

We will now describe each step, how to do it, and why.

## Take a Backup

Before doing any major system maintenance operation, it's crucial to have an up-to-date backup, ensuring that if anything goes wrong, there is a point in time to return to for the database's state.

In addition, all operations should be tested on a staging or a production-like environment as a "dry run" before attempting this on application-critical systems.   Performing backups is covered in the user guide in this repository.

## Scale the Core Statefulset Up

If you'd normally have 3 core members in your statefulset, they are providing a valuable 
[high availability purpose, and a quorum](https://neo4j.com/docs/operations-manual/current/clustering/introduction/#causal-clustering-introduction-operational).

In a rolling upgrade operation, we are going to take each server *down* in its turn.  And while one is stopping/restarting, we're (temporarily) damaging the HA characteristics of the
cluster, reducing it's ability to serve queries.   To mitigate this, before doing
a rolling upgrade we scale the cluster *up*, from say 3 cores to 5.  We will then roll
changes through - we will at any given moment have 4 of 5 cores available.

Given a cluster deployment named "mygraph", you can scale it to 5 cores like so:

```
kubectl scale statefulsets mygraph-neo4j-core --replicas=5
```

This should immediately schedule 2 new pods with the same configuration (and the *old* version of Neo4j) to start up and join the cluster.  It is recommended that you scale by 2, not by 1,
to ensure that the number of cores in the cluster is always odd.

> **REMEMBER** when new members join the cluster, before the cluster is stable, they
> need to pull current transactional state.  Having members restore from a recent
> backup first is strongly recommended, to minimize the load of the 
> [catch-up process](https://neo4j.com/docs/operations-manual/current/clustering-advanced/lifecycle/#causal-clustering-catchup-protocol).

If you do not restore from backups on each pod, it will still work, but the cluster may take
substantial time for the new members to catch up, particularly if you have a lot of data or
transactional history.

Consult [scaling statefulsets](https://kubernetes.io/docs/tasks/run-application/scale-stateful-set/#scaling-statefulsets) in the kubernetes documentation for more information.

## Choose and Apply your Update Strategy

For more details, consult the Kubernetes [Update strategies for statefulsets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#update-strategies).

In the next steps, we're going to tell the statefulset to change versions of Neo4j that it is
running.  Before we do, we need to give it a strategy of how the rolling upgrade should proceed.
You basically have 2 key options, `RollingUpdate` and `OnDelete`.  This document assumes the
`RollingUpdate` strategy.  In this situation, Kubernetes does the following:

* Starting from the end (the highest core number) it shuts a pod down, and restarts it.  Because
we're going to patch the version, when that pod restarts, it will come up with the new version.
* Kubernetes waits for the pod to get to the Ready state.
* The rolling update proceeds with the next lowest index.

### Criticality of Readiness / Liveness Checks

Kubernetes doesn't know much about Neo4j.  It needs to be provided a definiton of what it
means for Neo4j to be "ready".   If the new pod isn't ready before the next one rolls, you
could get into a situation where the cluster doesn't have time to stabilize before losing
more members.  

The readiness checks described in the user manual cover this purpose.  Review them for
adequacy and appropriateness for your deployment, and revise if necessary *before* performing
a rolling upgrade.

## Patch the StatefulSet

Up until now, everything we're doing was just preparation, but this step actually affects the
change.  If you run this, then subject to the `UpdateStrategy` you specified above, Kubernetes
will start killing & restarting pods.

```
NEW_VERSION=neo4j:4.1.0-enterprise
kubectl patch statefulset mygraph-neo4j-core --type='json' \
    -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"'$NEW_VERSION'"}]'
```

## Monitor the Process

What we're looking for:
* Only one pod at a time stopping/restarting
* Consistent ability to maintain a connection to the cluster, with both read & write ability throughout.
* Monitor the `debug.log` file and container output to make sure the process is proceeding correctly

## (Optional) Apply the Process to the Read Replicas

Read replicas are handled in a second StatefulSet.  The process of applying an update
strategy, patching the statefulset, and monitoring is the same though.

**Recommended:** make sure you have a fully migrated & happy core cluster member set
before working with read replicas.  Do not roll both sets at once.

## Scale Back Down

If everything was completed correctly, you should end up with a 5 member cluster with
the new version, 100% online.  After success, the extra members are no longer needed,
and you may scale back down to the original size of the cluster.

```
kubectl scale statefulsets mygraph-neo4j-core --replicas=3
```

# Alternatives to Rolling Upgrades

If you can tolerate a period of write-unavailabilty while maintaining full read-availability,
Kubernetes provides a secondary option to a rolling upgrade.  This document will focus on
how to do rolling upgrades, but as a sketch of the main alternative:

1. Configure your current cluster to be backed by a single DNS record (mycluster.company.com)
2. Take a backup of your current cluster.
3. Launch a second cluster running the new version of Neo4j (mycluster-updated).  Restore
this cluster from the last backup of `mycluster`.
4. Hot swap the DNS record to point to the second cluster (mycluster-updated)
5. Shut down the original cluster (mycluster)

**Important** if you adopt this approach, you will need a maintenance window where you are
not accepting writes, as of the point of backup.  The period of write unavailability should
be between steps 2 and the readiness of DNS in step 4.  If writes come in during this time
period to the original cluster, they will be missing form the updated cluster.

This approach should maintain read availability throughout, and it is reasonably safe; i.e.
if the new cluster fails to migrate properly or there is a data issue, this does not compromise
the availability of the running production system.
