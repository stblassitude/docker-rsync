# Access data inside Docker via rsync

When running dockerized applications, it can be cumbersome to access their data from outside of Docker. Especially when using Kubernetes, accessing data stored in volumes can be hard to impossible from your local host.

This image helps you need to create, update, retrieve, or delete data that's stored inside volume, using [rsync](https://rsync.samba.org).

Containers and pods can be configured to have access to volumes. This image starts up rsync in such a way that you can use `docker run` or `kubectl run` to connect from your local machine to a container and use rsync to move data between your machine and the (possibly remote) Docker or k8s cluster.

It uses rsyncs option to supply a command that is used when connecting to a remote server. Instead of using `ssh`, you can run this Docker image in its place.

## Examples

### List contents of Docker volume

This command lists the contents of the Docker volume "my-data-volume". Note that the hostname `foo` is ignored by the image, but it needed to make rsync try and connect to the remote host using the command supplied.

```sh
rsync --rsh="docker run -i --rm -v my-data-volume:/data rsync" --list-only foo:
```

### Copy data from a local directory to a k8s volume

This example is slightly more involved, since k8s does not allow the ad-hoc execution of images in the way `docker run` provides. Instead, we need to start up a pod with the desired volume attached, and then run rsync. Finally, we stop the pod again.

#### Create pod

Contents of `rsync-pod.yaml`
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: rsync-test-data
spec:
  containers:
  - name: rsync
    image: stblassitude/rsync
    command: ["wrapper"]
    args: ["start"]
    volumeMounts:
    - name: test-data
      mountPath: /data
  volumes:
  - name: test-data
    persistentVolumeClaim:
      claimName: test-data
```

Note the `start` argument which will make the image sleep in an endless loop. This is required so we can run the rsync command inside the running pod.

The pod attached the volume claim `test-data` under `/data`, where it will be available through rsync.

If you need to access the volume as a specific user, add a [securityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod) specification to the pod definition.

```
echo kubectl apply -f rsync-pod.yaml
```

Note that depending on your cluster and availability of resource, starting the pod might take a minute.

If the volume you're accessing cannot be shared between pods (for example, it's `accessModes` is set to `ReadWriteOnce`), and that volume is currently attached to another pod, the `rsync` pod might not start until the other pod is deleted.

### Copy data

Run whatever rsync command is necessary to copy data into or out of the volume. For example, to completely replicate the contents of the local `test-data` directory to the volume, you can use the following rsync command:

```sh
rsync --rsh="kubectl exec -i rsync wrapper" -a test-data foo:.
```

### Shut down the pod

In order for other pods to gain access to the volume, it might be necessary to shut down the rsync pod after copying the data.

```
kubectl delete pod rsync-test-data
```
