---
title: "Persistent storage"
weight: 8
---

By default, data in containers is not persistent as was the case e.g. in {{<link "attaching-a-database">}}. This means that the data written in a container is lost as soon as it does not exist anymore. We want to prevent this from happening. One possible solution to this problem is to use persistent storage.


## Request storage

Attaching persistent storage to a Pod happens in two steps. The first step includes the creation of a so-called _PersistentVolumeClaim_ (PVC) in our namespace. This claim defines amongst other things what size we would like to get.

The PersistentVolumeClaim only represents a request but not the storage itself. It is automatically going to be bound to a _PersistentVolume_ by {{% param distroName %}}, one that has at least the requested size. If only volumes exist that have a bigger size than was requested, one of these volumes is going to be used. The claim will automatically be updated with the new size. If there are only smaller volumes available, the claim cannot be fulfilled as long as no volume with the exact same or larger size is created.


## Attaching a volume to a Pod

{{% onlyWhenNot openshift %}}
In a second step, the PVC from before is going to be attached to the Pod. In {{<link "scaling">}} we edited the deployment configuration in order to insert a readiness probe. We are now going to do the same for inserting the persistent volume.
{{% /onlyWhenNot %}}
{{% onlyWhen openshift %}}
In a second step, the PVC from before is going to be attached to the Pod. In {{<link "scaling">}} we used `{{% param cliToolName %}} set` to add a readiness probe to the Deployment. We are now going to do the same and insert the PersistentVolume.
{{% /onlyWhen %}}


## {{% task %}} Add a PersistentVolume

{{% onlyWhen openshift %}}
The `oc set volume` command makes it possible to create a PVC and attach it to a Deployment in one fell swoop:

{{% alert title="Note" color="info" %}}
If you are using Windows, your shell might assume that it has to use the POSIX-to-Windows path conversion for the mount path `/var/lib/mysql`.
PowerShell is known to not do this while, e.g., Git Bash does.

Prepend your command with `MSYS_NO_PATHCONV=1` if the resulting mount path was mistakenly converted.
{{% /alert %}}

{{% onlyWhenNot baloise %}}
```bash
oc set volume dc/mariadb --add --name=mariadb-data --claim-name=mariadb-data --type persistentVolumeClaim --mount-path=/var/lib/mysql --claim-size=1G --overwrite --namespace <namespace>
```
{{% /onlyWhenNot %}}

{{% onlyWhen baloise %}}
```bash
oc set volume deploy/mariadb --add --name=mariadb-data --claim-name=mariadb-data --type persistentVolumeClaim --mount-path=/var/lib/mysql --claim-size=1G --overwrite --namespace <namespace>
```
{{% /onlyWhen %}}

With the instruction above we create a PVC named `mariadb-data` of 1Gi in size, attach it to the DeploymentConfig `mariadb` and mount it at `/var/lib/mysql`. This is where the MariaDB process writes its data by default so after we make this change, the database will not even notice that it is writing in a PersistentVolume.
{{% /onlyWhen %}}
{{% onlyWhen openshift %}}
{{% alert title="Note" color="info" %}}
Because we just changed the DeploymentConfig with the `oc set` command, a new Pod was automatically redeployed. This unfortunately also means that we just lost the data we inserted before.
{{% /alert %}}
{{% /onlyWhen %}}

{{% onlyWhenNot openshift %}}
The following command creates a PersistentVolumeClaim which requests a volume of 1Gi size.
Save it to `pvc.yaml`:

{{< readfile file="/content/en/docs/persistent-storage/pvc.yaml" code="true" lang="yaml" >}}

And create it with:

```bash
kubectl apply -f pvc.yaml --namespace <namespace>
```

We now have to insert the volume definition in the correct section of the MariaDB deployment:

```bash
kubectl edit deployment mariadb --namespace <namespace>
```

Add both parts `volumeMounts` and `volumes`

```yaml
...
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        # start to copy here
        volumeMounts:
        - name: mariadb-data
          mountPath: /var/lib/mysql
      volumes:
      - name: mariadb-data
        persistentVolumeClaim:
          claimName: mariadb-data
      # stop to copy here
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
...
```

{{% /onlyWhenNot %}}
{{% onlyWhenNot openshift %}}
{{% alert title="Note" color="info" %}}
Because we just changed the Deployment a new Pod was automatically redeployed. This unfortunately also means that we just lost the data we inserted before.
{{% /alert %}}
{{% /onlyWhenNot %}}

We need to redeploy the application pod, our application automatically creates the database schema at startup time. Wait for the database pod to be started fully before restarting the application pod.

If you want to force a redeployment of a Pod, you can use this:

```bash
{{% param cliToolName %}} rollout restart deployment example-web-app --namespace <namespace>
```

Using the command `{{% param cliToolName %}} get persistentvolumeclaim` or `{{% param cliToolName %}} get pvc`, we can display the freshly created PersistentVolumeClaim:

```bash
{{% param cliToolName %}} get pvc --namespace <namespace>
```

Which gives you an output similar to this:

```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mariadb-data     Bound    pvc-2cb78deb-d157-11e8-a406-42010a840034   1Gi        RWO            standard       11s
```

The two columns `STATUS` and `VOLUME` show us that our claim has been bound to the PersistentVolume `pvc-2cb78deb-d157-11e8-a406-42010a840034`.


## Error case

If the container is not able to start it is the right moment to debug it!
Check the logs from the container and search for the error.

```bash
{{% param cliToolName %}} logs mariadb-f845ccdb7-hf2x5 --namespace <namespace>
```

{{% alert title="Note" color="info" %}}
If the container won't start because the data directory already has files in it, use the `{{% onlyWhenNot openshift %}}kubectl exec{{% /onlyWhenNot %}}{{% onlyWhen openshift %}}oc debug{{% /onlyWhen %}}` command mentioned in {{<link "attaching-a-database">}} to check its content and remove it if necessary.{{% /alert %}}


## {{% task %}} Persistence check


### Restore data

Repeat [the task to import a database dump](../attaching-a-database/#task-75-import-a-database-dump).


### Test

Scale your MariaDB Pod to 0 replicas and back to 1. Observe that the new Pod didn't loose any data.
