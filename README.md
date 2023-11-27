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
command line tools or API. In this example we use the ramen testing
environment.

### Setting up the testing environment

*Ramen* test environment creates 3 *minikube* clusters (hub, dr1, dr2)
on your laptop, including *OCM*, *Rook*, *Kubevirt*, *CDI*, and many
other components required by *Ramen*.

See the [User quick start guide](https://github.com/RamenDR/ramen/blob/main/docs/user-quick-start.md)
for complete instructions.

> [!IMPORTANT]
> When starting the environment, use the `regional-dr-kubvirt.yaml`
> configuration.

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

At this point *Ramen* controls the VM, and you can test DR
actions.

### Failing over to another cluster

In case of disaster you can force the VM to run on the other cluster.
The VM will start on the other cluster using the data from the last
replication. Data since the last replication is lost.

Patch the VM `DRPlacementControl` resource to set `action` and
`failoverCluster`:

```sh
kubectl patch drpc kubevirt-drpc \
    --patch '{"spec": {"action": "Failover", "failoverCluster": "dr2"}}' \
    --type merge \
    --namespace kubevirt-sample \
    --context hub
```

The VM will start on the failover cluster ("dr2"), and the instance
running on the original cluster ("dr1") will be terminated.

To wait until the operation is completed use:

```sh
kubectl wait drpc kubevirt-drpc \
    --for condition=PeerReady \
    --namespace kubevirt-sample \
    --timeout 5m \
    --context hub
```

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

To wait until the operation is completed, wait until the drpc phase is
`Relocated`:

```sh
kubectl wait drpc kubevirt-drpc \
    --for jsonpath='{.status.phase}=Relocated' \
    --namespace kubevirt-sample \
    --timeout 5m \
    --context hub
```

Finally wait for the `PeerReady` condition:

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
