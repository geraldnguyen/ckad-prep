# State Persistence

## Objectives
- Understand PersistentVolumeClaims for storage

## Q1. Create a PersistentVolume named `host-pv` of size 2Gi. Create a PersistentVolumeClaim of size 3Gi and another PersistentVolumeClaim of 1Gi. Verify that only the later reach "bound" status

<details><summary>Solution</summary>

Create `host-pv` PersistentVolume of size 2Gi

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: host-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/mnt/data"
```

Create a PersistentVolumeClaim of 3Gi

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-claim-3gi
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
```

Create a PersistentVolumeClaim of 1Gi

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-claim-1gi
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

Verify that only the 1Gi claim is served

```
$ k get pv,pvc
NAME                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
persistentvolume/host-pv   2Gi        RWX            Retain           Bound    default/pv-claim-1gi   manual                  3m52s

NAME                                 STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/pv-claim-1gi   Bound     host-pv   2Gi        RWX            manual         8s
persistentvolumeclaim/pv-claim-3gi   Pending                                       manual         65s
```


</details>

## See also [05 - Pod design #Q1](./05-pod-design.md#q1-create-a-job-that-downloads-multiple-files-from-a-configdownloadlist-specified-in-a-persistentvolume-the-job-saves-download-files-to-the-same-volume-under-download-directory)
