# ocm-kubevirt-samples

This repo contains sample kubevirt resources for deployment in
Kubernetes or OpenShift clusters.


## Using in OpenShift

Use ACM console to deploy the VM, enable DR or delete the
VM.

### Deploying the VM

1. Create new application using:
   - repo: this repo URL
   - branch: main
   - path: one of the `vm-*-odr-*` directories

1. To enable DR assign DR Policy

### Testing DR

Use ACM console to perform `failover` or `relocate` actions.

### Deleting the VM

Use ACM console to delete the application.


## Using in Kubernetes

In Kubernetes you need to deploy the VM and DR resources using the
command line tools or API.

### Adding your SSH key

If you want to inject your SSH key into the VM, you need to replace
the included SSH public key with your own public key.

1. Fork this repo in github
1. Replace the contents of the `vm-standalone-pvc-k8s/test_rsa.pub` with
   *your* public key (e.g. `~/.ssh/id_rsa.pub`)
1. Update `channel/channel.yaml` to point to *your* repository
   (e.g. https://github.com/my-github-user/ocm-kubevirt-samples.git)
1. If you are not using the `main` branch update
   `subscription/subscription.yaml` to point to the right branch.

### Deploying the channel

Deploy the channel to introduce this repo to *OCM*:

```sh
kubectl apply -k channel --context hub
```

### Deploying the VM subscription

To start the VM deploy the subscription:

```sh
kubectl apply -k subscription --context hub
```

The subscription starts the VM `vm-standalone-pvc-k8s` on one of the
clusters in the default clusterset.

The subscription is optimized for *Ramen* minikube based test
environment. To use in another setup you may need to modify the
resources.

### Inspecting the VM status

To find where the VM was placed look at the PlacementDecisions status:

```sh
kubectl get placementdecisions -n kubevirt-sample --context hub \
    -o jsonpath='{.items[0].status.decisions[0].clusterName}{"\n"}'
```

To inspect the VM and DR resources use:

```sh
watch -n 5 kubectl get vm,vmi,pod,pvc,vrg,vr -n kubevirt-sample --context dr1
```

Example output:

```console
Every 5.0s: kubectl get vm,vmi,pod,pvc,vrg,vr -n kubevirt-sample --context dr1

NAME                                   AGE     STATUS    READY
virtualmachine.kubevirt.io/sample-vm   5m51s   Running   True

NAME                                           AGE     PHASE     IP            NODENAME   READY
virtualmachineinstance.kubevirt.io/sample-vm   5m51s   Running   10.244.0.61   dr1        True

NAME                                READY   STATUS    RESTARTS   AGE
pod/virt-launcher-sample-vm-chsrh   1/1     Running   0          5m51s

NAME                                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
persistentvolumeclaim/sample-vm-pvc   Bound    pvc-149bd414-c1b8-443c-90f7-7e30fc519eb4   128Mi      RWX            rook-ceph-block   5m51s
```

At this point there there are no `vrg` and `vr` resources, since we did
not enable DR for the VM yet.

We can inspect the underlying RBD image backing the VM PVC using the
`rook-ceph` kubectl krew plugin:

```
kubectl rook-ceph --context dr1 rbd du -p replicapool
```

Example output:

```console
NAME                                          PROVISIONED  USED
csi-vol-a3fdb384-2e31-49f1-bd48-a97b2d79f981      128 MiB  80 MiB
```

### Enabling DR for the VM

To allow *Ramen* to protect the VM, you need to disable *OCM*
scheduling by adding an annotation to the VM placement:

```sh
kubectl annotate placement kubevirt-placement \
    cluster.open-cluster-management.io/experimental-scheduling-disable=true \
    --namespace kubevirt-sample \
    --context hub
```

Deploy the DR resources to enable DR:

```sh
kubectl apply -k dr --context hub
```

At this point *Ramen* controls the VM placement and protects the VM data
by replicating it to the secondary cluster ("dr2").

To wait until the VM data is replicating to the secondary cluster, wait
for the `PeerReady` condition:

```sh
kubectl wait drpc kubevirt-drpc \
    --for condition=PeerReady \
    --namespace kubevirt-sample \
    --timeout 5m \
    --context hub
```

### Inspecting the VM DR status

We can inspect the VM DR status using the `DRPlacementControl` resource
on the hub cluster:

```sh
watch -n 5 kubectl get drpc -n kubevirt-sample --context hub -o wide
```

Example output:

```console
Every 5.0s: kubectl get drpc -n kubevirt-sample --context hub -o wide

NAME            AGE   PREFERREDCLUSTER   FAILOVERCLUSTER   DESIREDSTATE   CURRENTSTATE   PROGRESSION   START TIME             DURATION       PEER READY
kubevirt-drpc   51s   dr1                                                 Deployed	 Completed     2023-11-19T20:26:53Z   5.035609263s   True
```

To get more details we can watch the VM and DR reosurces on the managed
cluster:

```sh
watch -n 5 kubectl get vm,vmi,pod,pvc,vrg,vr -n kubevirt-sample --context dr1
```

Example output:

```console
Every 5.0s: kubectl get vm,vmi,pod,pvc,vrg,vr -n kubevirt-sample --context dr1

NAME                                   AGE   STATUS    READY
virtualmachine.kubevirt.io/sample-vm   16m   Running   True

NAME                                           AGE   PHASE     IP            NODENAME   READY
virtualmachineinstance.kubevirt.io/sample-vm   16m   Running   10.244.0.61   dr1        True

NAME                                READY   STATUS    RESTARTS   AGE
pod/virt-launcher-sample-vm-chsrh   1/1     Running   0          16m

NAME                                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
persistentvolumeclaim/sample-vm-pvc   Bound    pvc-149bd414-c1b8-443c-90f7-7e30fc519eb4   128Mi      RWX            rook-ceph-block   16m

NAME                                                        DESIREDSTATE   CURRENTSTATE
volumereplicationgroup.ramendr.openshift.io/kubevirt-drpc   primary        Primary

NAME                                                               AGE     VOLUMEREPLICATIONCLASS   PVCNAME         DESIREDSTATE   CURRENTSTATE
volumereplication.replication.storage.openshift.io/sample-vm-pvc   2m46s   vrc-sample               sample-vm-pvc   primary        Primary
```

The cirros VM used by this example includes a logger service logging a
message every 10 seconds:

```sh
virtctl ssh cirros@sample-vm -n kubevirt-sample --known-hosts= --context dr1 -c 'head /var/log/ramen.log'
```

Example output:

```console
Sun Nov 19 20:13:20 UTC 2023 START uptime=3.33
Sun Nov 19 20:13:30 UTC 2023 UPDATE
Sun Nov 19 20:13:40 UTC 2023 UPDATE
Sun Nov 19 20:13:50 UTC 2023 UPDATE
Sun Nov 19 20:14:00 UTC 2023 UPDATE
Sun Nov 19 20:14:10 UTC 2023 UPDATE
Sun Nov 19 20:14:20 UTC 2023 UPDATE
Sun Nov 19 20:14:30 UTC 2023 UPDATE
Sun Nov 19 20:14:40 UTC 2023 UPDATE
Sun Nov 19 20:14:50 UTC 2023 UPDATE
```

Ramen set up RBD mirroring for the underlying RBD image. The same image
is created on the secondary cluster ("dr2"):

```
kubectl rook-ceph --context dr2 rbd du -p replicapool
```

Example output:

```console
NAME                                          PROVISIONED  USED
csi-vol-a3fdb384-2e31-49f1-bd48-a97b2d79f981      128 MiB  80 MiB
```

In case of a disaster in the primary cluster, we can start the VM using
the replica on the secondary cluster.

### Failing over to another cluster

In case of disaster you can force the VM to run on the other cluster.
The VM will start on the other cluster using the data from the last
replication. Data since the last replication is lost.

To simulate a disaster we can pause the minkube VM running cluster
`dr1`:

```sh
virsh -c qemu:///system suspend dr1
```

To start `Failover` action, patch the VM `DRPlacementControl` resource
to set `action` and `failoverCluster`:

```sh
kubectl patch drpc kubevirt-drpc \
    --patch '{"spec": {"action": "Failover", "failoverCluster": "dr2"}}' \
    --type merge \
    --namespace kubevirt-sample \
    --context hub
```

The VM will start on the failover cluster ("dr2"). Nothing will change
on the primary cluster ("dr1") since it is still paused.

Inspecting the `/var/log/ramen.log` via SSH show how much data was lost
during the failover:

```sh
virtctl ssh cirros@sample-vm -n kubevirt-sample --known-hosts= --context dr2 -c 'tail -f /var/log/ramen.log'
```

Example output:

```console
Sun Nov 19 20:33:42 UTC 2023 UPDATE
Sun Nov 19 20:33:53 UTC 2023 UPDATE
Sun Nov 19 20:34:03 UTC 2023 UPDATE
Sun Nov 19 20:34:13 UTC 2023 UPDATE
Sun Nov 19 20:34:23 UTC 2023 UPDATE
Sun Nov 19 20:34:33 UTC 2023 UPDATE
Sun Nov 19 20:34:43 UTC 2023 UPDATE
Sun Nov 19 20:34:53 UTC 2023 UPDATE
Sun Nov 19 20:37:43 UTC 2023 START uptime=3.18
Sun Nov 19 20:37:53 UTC 2023 UPDATE
Sun Nov 19 20:38:03 UTC 2023 UPDATE
```

To enable replication from the secondary cluster to the primary cluster,
we need to recover the primary cluster. In this example we can resume
the minikube VM:

```sh
virsh -c qemu:///system resume dr1
```

*Ramen* will clean up the VM resources from the primary cluster and
enable RBD mirroring from the secondary cluster ("dr2") to the primary
cluster ("dr1").

To wait until the VM data is replicated again to the other cluster:

```sh
kubectl wait drpc kubevirt-drpc \
    --for condition=PeerReady \
    --namespace kubevirt-sample \
    --timeout 5m \
    --context hub
```

Since the primary cluster is recovered, we can move the VM back to the
primary cluster.

### Relocating to another cluster

To move the VM back to the original cluster after a disaster
you can use the `Relocate` action. The VM will be terminated on
the current cluster, and started on the other cluster. No data is lost
during this operation.

Patch the VM `DRPlacementControl` resource to set `action` and
if needed, `preferredCluster`.

```sh
kubectl patch drpc kubevirt-drpc \
    --patch '{"spec": {"action": "Relocate", "preferredCluster": "dr1"}}' \
    --type merge \
    --namespace kubevirt-sample \
    --context hub
```

The VM will terminate on the current cluster and will start on the
preferred cluster.

To wait until the VM is relocated to the primary cluster, wait until the
drpc phase is `Relocated`:

```sh
kubectl wait drpc kubevirt-drpc \
    --for jsonpath='{.status.phase}=Relocated' \
    --namespace kubevirt-sample \
    --timeout 5m \
    --context hub
```

Inspecting `/var/log/ramen.log` shows that the VM was terminated cleanly
on the secondary cluster ("dr2") and started on the primary cluster
("dr1"). No data was lost!

```sh
virtctl ssh cirros@sample-vm -n kubevirt-sample --known-hosts= --context dr1 -c 'tail -f /var/log/ramen.log'
```

Example output:

```console
Sun Nov 19 20:41:53 UTC 2023 UPDATE
Sun Nov 19 20:42:03 UTC 2023 UPDATE
Sun Nov 19 20:42:13 UTC 2023 UPDATE
Sun Nov 19 20:42:23 UTC 2023 UPDATE
Sun Nov 19 20:42:33 UTC 2023 UPDATE
Sun Nov 19 20:42:43 UTC 2023 UPDATE
Sun Nov 19 20:42:53 UTC 2023 UPDATE
Sun Nov 19 20:42:58 UTC 2023 STOP
Sun Nov 19 20:41:17 UTC 2023 START uptime=3.45
Sun Nov 19 20:41:27 UTC 2023 UPDATE
Sun Nov 19 20:41:37 UTC 2023 UPDATE
```

To wait until the VM is replicating data again to the secondary cluster,
wait for the `PeerReady` condition:

```sh
kubectl wait drpc kubevirt-drpc \
    --for condition=PeerReady \
    --namespace kubevirt-sample \
    --timeout 5m \
    --context hub
```

### Disable DR for the VM

Delete the `dr` resources to disable DR:

```sh
kubectl delete -k dr --context hub
```

Enable *OCM* scheduling again by deleting the placement annotation we
added before:

```sh
kubectl annotate placement kubevirt-placement \
    cluster.open-cluster-management.io/experimental-scheduling-disable- \
    --namespace kubevirt-sample \
    --context hub
```

At this point *OCM* controls the VM and the storage used for replicating
the VM data on the DR clusters will be reclaimed.

### Undeploying the VM

Delete the subscription to stop and delete the VM:

```sh
kubectl delete -k subscription --context hub
```

### Delete the channel

When done you can delete the channel:

```sh
kubectl delete -k channel --context hub
```
