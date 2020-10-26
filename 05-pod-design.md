
# Pod Design (20%)

## Objectives
- Understand deployments and how to perform rolling updates
- Understand deployments and how to perform rollbacks
- Understasnd Jobs and CronJobs
- Understand how to use Labels, Selectors and Annotations

## Q1. Create a job that downloads multiple files from a `/config/download.list` specified in a PersistentVolume. The job saves download files to the same volume under `/download` directory

<details><summary>Solution</summary>

We need an PersistentVolume to store file downloaded by different jobs. Note that PersistentVolume is a cross-namespace resource

```
$ cat /tmp/kpv/pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-volume
  labels:
    type: pv-volume
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/tmp/kpv"
    
$ k apply -f /tmp/kpv/pv.yaml 
persistentvolume/pv-volume created
```

Let's next create the `download.list`

```
$ cat /tmp/download.list 
https://github.com/cncf/curriculum/blob/master/CKAD_Curriculum_V1.19.pdf
https://d33wubrfki0l68.cloudfront.net/eb4e41f2cba0cbc8d119f8d0eb2bd6935cb78fc8/ba7d6/images/community/kubernetes-community-final-02.jpg

$ k create configmap download-list --from-file=/tmp/download.list -n ckad
configmap/download-list created
```

Then finally the job and a claim to the persistent volume created earlier

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: linear-download
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      volumes:
      - name: config-volume
        configMap: 
          name: download-list
      - name: pv-volume
        persistentVolumeClaim:
          claimName: pv-claim 
      containers:
      - image: busybox
        name: linear-download
        resources: {}
        command:
        - sh
        - -c
        - cat /config/download.list | while read line; do echo Downloading $line; wget -O /download/$(basename $line) $line; done;
        volumeMounts:
        - name: config-volume
          mountPath: /config
        - name: pv-volume
          mountPath: /download
      restartPolicy: Never
status: {}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-claim
spec:
  storageClassName: manual
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

Verify that the job has finished

```
$ k get job,pods -n ckad --show-labels
NAME                        COMPLETIONS   DURATION   AGE     LABELS
job.batch/linear-download   1/1           7s         4m14s   controller-uid=e93c6dc8-0762-4aab-bc0e-d45f65ca51c5,job-name=linear-download

NAME                        READY   STATUS      RESTARTS   AGE     LABELS
pod/linear-download-zdgcg   0/1     Completed   0          4m14s   controller-uid=e93c6dc8-0762-4aab-bc0e-d45f65ca51c5,job-name=linear-download


$ k logs -l job-name=linear-download -n ckad
wget: note: TLS certificate validation not implemented
saving to '/download/CKAD_Curriculum_V1.19.pdf'
CKAD_Curriculum_V1.1 100% |********************************| 85594  0:00:00 ETA
'/download/CKAD_Curriculum_V1.19.pdf' saved
Downloading https://d33wubrfki0l68.cloudfront.net/eb4e41f2cba0cbc8d119f8d0eb2bd6935cb78fc8/ba7d6/images/community/kubernetes-community-final-02.jpg
Connecting to d33wubrfki0l68.cloudfront.net (52.84.228.202:443)
wget: note: TLS certificate validation not implemented
saving to '/download/kubernetes-community-final-02.jpg'
kubernetes-community 100% |********************************|  169k  0:00:00 ETA
'/download/kubernetes-community-final-02.jpg' saved

```
 

(Optional) Inspect the PersistentVolume from another pod to make sure the downloaded files are really there: 

```
$ cat /tmp/pv-inspect.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod
  name: pv-inspector
spec:
  volumes:
  - name: pv-volume
    persistentVolumeClaim:
      claimName: pv-claim 
  containers:
  - image: alpine
    name: pv-inspector
    command: 
    - sh
    - -c
    - sleep 1h
    resources: {}
    volumeMounts:
    - name: pv-volume
      mountPath: /download
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

$ k exec pv-inspector -n ckad -it -- sh
/ # ls -l /download/
total 256
-rw-r--r--    1 root     root         85594 Oct 13 17:36 CKAD_Curriculum_V1.19.pdf
-rw-r--r--    1 root     root        173219 Oct 13 17:36 kubernetes-community-final-02.jpg
/ # exit

```

</details> 

## Q2. Create multiple jobs which can download multiple (big) files in parallel. Follow the approach documented at https://kubernetes.io/docs/tasks/job/parallel-processing-expansion/#create-jobs-based-on-a-template

<details><summary>Solution</summary>
  
Create the job templates:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-download-$ITEM
  labels:
    download: parallel
spec:
  template:
    metadata:
      name: download-job
      labels:
        download: parallel
    spec:
      volumes:
      - name: pv-volume
        persistentVolumeClaim:
          claimName: pv-claim 
      containers:
      - name: busybox
        image: busybox
        command: ["sh", "-c", "FILE=$FILE_URL; echo Downloading $FILE; wget -O /download/$(basename $FILE) $FILE; sleep 5m"]
        volumeMounts:
        - name: pv-volume
          mountPath: /download
      restartPolicy: Never
```

Generate worker jobs for each files to be downloaded. Notes:
- We need to generate a job name that is a valid DNS-1123 subdomain. The `sed -e "s/_/-/g"` and `awk '{print tolower($0)}'` do the trick at least for the examples here
- We reuse the `download.list` file in earlier question

```bash
for file in `cat /tmp/download.list `; do filename=`echo $(basename $file) | sed -e "s/_/-/g" | awk '{print tolower($0)}'`; cat job-parallel-tmpl.yaml | sed "s/\$ITEM/$filename/" | sed "s#\$FILE_URL#$file#"> jobs/job-$filename.yaml; done
```

Execute the jobs

```
$ k create -f jobs -n ckad
job.batch/parallel-download-ckad-curriculum-v1.19.pdf created
job.batch/parallel-download-kubernetes-community-final-02.jpg created
```

</details>   

## Q3. Create a job that probe a website (e.g. localhost) on multiple ports. The job should runs in parallel (e.g. 3) with a `restartPolicy=OnFailure` and completions of 4 (e.g. 80, 8080, 3000, 1313)

<details><summary>Solution</summary>
  
Let's simulate a server

```
$ k run nginx --image=nginx -n ckad
pod/nginx created

$ k get pod nginx -n ckad -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          26s   172.17.0.6   minikube   <none>           <none>
``` 

We need a configuration ConfigMap:

```
$ k create configmap job-config --from-literal=host=172.17.0.6  --from-literal="ports=80 1313 3000 8080" -n ckad
configmap/job-config created
```

Create a job yaml with the usual PVC mount and volume's mountPath and a command to pick a port and probe it. For simplicity, we'll consider the present of files with port number as name as the coordination mechanism between pods. The following command will thus be sufficient: `for port in $ports; do test -f "/port-scan/$host/$port" || break; done; echo "Claimed port $port from $(hostname)"; touch "/port-scan/$host/$port";`. The command should also end with 0 exit code regardless of whether the port is open or not - we will use `sleep 1` to force a 0 exit code. The final yaml file should look similar to below:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: parallel-download
spec:
  completions: 4
  parallelism: 3
  template:
    metadata:
      creationTimestamp: null
    spec:
      volumes:
      - name: pv-volume
        persistentVolumeClaim:
          claimName: pv-claim
      containers:
      - image: busybox
        name: parallel-download
        resources: {}
        command:
        - sh
        - -c
        - mkdir -p /port-scan/$host; ls /port-scan/$host; for port in $ports; do test -f "/port-scan/$host/$port" || break; done; echo "Claimed port $port from $(hostname)"; touch "/port-scan/$host/$port"; wget -O "/port-scan/$host/$port" "$host:$port"; sleep 1
        volumeMounts:
        - name: pv-volume
          mountPath: /port-scan
        envFrom:
        - configMapRef:
            name: job-config
      restartPolicy: Never
status: {}
```
  
Run the jobs and monitor its pods' statuses and logs

```
$ k create -f parallel-job.yaml -n ckad
job.batch/parallel-download created

$ k get all -n ckad --show-labels
NAME                          READY   STATUS              RESTARTS   AGE   LABELS
pod/alpine                    1/1     Running             0          39h   run=alpine
pod/nginx                     1/1     Running             0          14h   run=nginx
pod/parallel-download-2cgj7   0/1     ContainerCreating   0          3s    controller-uid=aca2afbb-c006-4b28-bd12-e2f220272b4d,job-name=parallel-download
pod/parallel-download-f8qqp   0/1     ContainerCreating   0          3s    controller-uid=aca2afbb-c006-4b28-bd12-e2f220272b4d,job-name=parallel-download
pod/parallel-download-kfdxd   0/1     ContainerCreating   0          3s    controller-uid=aca2afbb-c006-4b28-bd12-e2f220272b4d,job-name=parallel-download
pod/pv-inspector              1/1     Running             0          14h   run=pod

NAME                          COMPLETIONS   DURATION   AGE   LABELS
job.batch/parallel-download   0/4           3s         3s    controller-uid=aca2afbb-c006-4b28-bd12-e2f220272b4d,job-name=parallel-download


$ k get all -n ckad --show-labels
NAME                          READY   STATUS      RESTARTS   AGE   LABELS
pod/alpine                    1/1     Running     0          39h   run=alpine
pod/nginx                     1/1     Running     0          14h   run=nginx
pod/parallel-download-2cgj7   0/1     Completed   0          24s   controller-uid=aca2afbb-c006-4b28-bd12-e2f220272b4d,job-name=parallel-download
pod/parallel-download-f8qqp   0/1     Completed   0          24s   controller-uid=aca2afbb-c006-4b28-bd12-e2f220272b4d,job-name=parallel-download
pod/parallel-download-gg5js   0/1     Completed   0          17s   controller-uid=aca2afbb-c006-4b28-bd12-e2f220272b4d,job-name=parallel-download
pod/parallel-download-kfdxd   0/1     Completed   0          24s   controller-uid=aca2afbb-c006-4b28-bd12-e2f220272b4d,job-name=parallel-download
pod/pv-inspector              1/1     Running     0          14h   run=pod

NAME                          COMPLETIONS   DURATION   AGE   LABELS
job.batch/parallel-download   4/4           23s        24s   controller-uid=aca2afbb-c006-4b28-bd12-e2f220272b4d,job-name=parallel-download


$ k logs -l job-name=parallel-download -n ckad
Connecting to 172.17.0.6:8080 (172.17.0.6:8080)
1313
3000
80
Claimed port 8080 from parallel-download-2cgj7
wget: can't connect to remote host (172.17.0.6): Connection refused
80
Claimed port 1313 from parallel-download-f8qqp
Connecting to 172.17.0.6:1313 (172.17.0.6:1313)
wget: can't connect to remote host (172.17.0.6): Connection refused
1313
80
Claimed port 3000 from parallel-download-gg5js
Connecting to 172.17.0.6:3000 (172.17.0.6:3000)
wget: can't connect to remote host (172.17.0.6): Connection refused
Claimed port 80 from parallel-download-kfdxd
Connecting to 172.17.0.6:80 (172.17.0.6:80)
saving to '/port-scan/172.17.0.6/80'
80                   100% |********************************|   612  0:00:00 ETA
'/port-scan/172.17.0.6/80' saved

```
  
</details>  

## Q4. Create a firedrill job that runs on a particular date, hour, minute and trigger a PA (e.g. Please exit the building via nearest staircase) every specified interval. Keep up to 5 last runs in history

<details><summary>Solution</summary>

Create a CronJob schedule to run at 5 minutes from <now>, interval 2 seconds. Note: because default minikube timezone is UTC, the schedule below is adjusted to UTC timezone.
  
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  creationTimestamp: null
  name: firedrill
spec:
  successfulJobsHistoryLimit: 5
  jobTemplate:
    metadata:
      creationTimestamp: null
      name: firedrill
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - command:
            - sh
            - -c
            - echo $(date) Please exit the building via the nearesst staircase
            image: busybox
            name: firedrill
            resources: {}
          restartPolicy: OnFailure
  schedule: 3/2 1 16 10 ?
status: {}
```
  
```
$ k get cj -n ckad --show-labels --watch
NAME        SCHEDULE        SUSPEND   ACTIVE   LAST SCHEDULE   AGE   LABELS
firedrill   3/2 1 16 10 ?   False     0        <none>          21s   <none>
firedrill   3/2 1 16 10 ?   False     1        4s              91s   <none>
firedrill   3/2 1 16 10 ?   False     0        14s             101s   <none>
firedrill   3/2 1 16 10 ?   False     1        4s              3m31s   <none>
firedrill   3/2 1 16 10 ?   False     0        14s             3m41s   <none>

$ k logs -l job-name -n ckad
Fri Oct 16 01:03:10 UTC 2020 Please exit the building via the nearesst staircase
Fri Oct 16 01:05:11 UTC 2020 Please exit the building via the nearesst staircase

$ k get job -n ckad --watch
NAME                   COMPLETIONS   DURATION   AGE
firedrill-1602783120   0/1                      0s
firedrill-1602783120   0/1           0s         0s
firedrill-1602783120   1/1           8s         8s
firedrill-1602783240   0/1                      0s
firedrill-1602783240   0/1           1s         1s
firedrill-1602783240   1/1           7s         7s
^C$

$k get pod -n ckad --show-labels --watch
NAME                         READY   STATUS    RESTARTS   AGE   LABELS
firedrill-1602783120-4mj5h   0/1     Pending   0          0s    controller-uid=c43b082d-5df7-40db-b857-2d6bb52fccb7,job-name=firedrill-1602783120
firedrill-1602783120-4mj5h   0/1     Pending   0          0s    controller-uid=c43b082d-5df7-40db-b857-2d6bb52fccb7,job-name=firedrill-1602783120
firedrill-1602783120-4mj5h   0/1     ContainerCreating   0          0s    controller-uid=c43b082d-5df7-40db-b857-2d6bb52fccb7,job-name=firedrill-1602783120
firedrill-1602783120-4mj5h   0/1     Completed           0          8s    controller-uid=c43b082d-5df7-40db-b857-2d6bb52fccb7,job-name=firedrill-1602783120
firedrill-1602783240-srxmj   0/1     Pending             0          0s    controller-uid=9030fe88-4d3e-4b34-888b-5005d77dc277,job-name=firedrill-1602783240
firedrill-1602783240-srxmj   0/1     Pending             0          0s    controller-uid=9030fe88-4d3e-4b34-888b-5005d77dc277,job-name=firedrill-1602783240
firedrill-1602783240-srxmj   0/1     ContainerCreating   0          1s    controller-uid=9030fe88-4d3e-4b34-888b-5005d77dc277,job-name=firedrill-1602783240
firedrill-1602783240-srxmj   0/1     Completed           0          7s    controller-uid=9030fe88-4d3e-4b34-888b-5005d77dc277,job-name=firedrill-1602783240
^C$

 k get job -n ckad --watch
NAME                   COMPLETIONS   DURATION   AGE
firedrill-1602810180   1/1           7s         2m12s
firedrill-1602810300   1/1           8s         12s

```

After some time, verify that up to 5 last jobs are present in the cron's history

```
$ k get jobs -n ckad --show-labels
NAME                   COMPLETIONS   DURATION   AGE     LABELS
firedrill-1602810180   1/1           7s         8m43s   controller-uid=00842d9b-e5db-4ee5-b3db-033872ba4d07,job-name=firedrill-1602810180
firedrill-1602810300   1/1           8s         6m43s   controller-uid=e90b6760-ee8b-4f24-9da8-427077eea1d7,job-name=firedrill-1602810300
firedrill-1602810420   1/1           6s         4m42s   controller-uid=3e3cc830-dc29-4f1f-b2ad-d242c89a49dc,job-name=firedrill-1602810420
firedrill-1602810540   1/1           6s         2m42s   controller-uid=3b3deff4-8cc6-4c7e-bc60-2be6c8a828dc,job-name=firedrill-1602810540
firedrill-1602810660   1/1           7s         42s     controller-uid=65468599-ea78-4d8f-913f-e8efd557bc48,job-name=firedrill-1602810660

$ k get jobs -n ckad --show-labels
NAME                   COMPLETIONS   DURATION   AGE     LABELS
firedrill-1602810300   1/1           8s         8m21s   controller-uid=e90b6760-ee8b-4f24-9da8-427077eea1d7,job-name=firedrill-1602810300
firedrill-1602810420   1/1           6s         6m20s   controller-uid=3e3cc830-dc29-4f1f-b2ad-d242c89a49dc,job-name=firedrill-1602810420
firedrill-1602810540   1/1           6s         4m20s   controller-uid=3b3deff4-8cc6-4c7e-bc60-2be6c8a828dc,job-name=firedrill-1602810540
firedrill-1602810660   1/1           7s         2m20s   controller-uid=65468599-ea78-4d8f-913f-e8efd557bc48,job-name=firedrill-1602810660
firedrill-1602810780   1/1           7s         19s     controller-uid=6e49bafe-35d2-4035-b70e-d7803669b992,job-name=firedrill-1602810780

```
 </details> 
 
 ## Q5. Create an deployment of 3 replicas for image `huypuma/springboot-helloapi` with additional annotation such as `purpose=ckad-prep`, `powered-by=springboot` and labels such as `env=test`, `app=hello-api`. Expose the deployment via a service and verify that it is working by making a curl call to `http://<service-ip>:8080/hello/<a-name>` and `kubectl logs` command on pods with mentioned labels
 
<details><summary>Solution</summary>
  
Create a deployment with 3 replicas, annotations `purpose: ckad-prep` and `power-by: springboot`, labels `env: test`, `app: helllo-api`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: hello-api
  name: hello-api
  namespace: ckad
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-api
      env: test
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      annotations:
        purpose: ckad-prep
        powered-by: springboot 
      labels:
        app: hello-api
        env: test
    spec:
      containers:
      - image: huypuma/springboot-helloapi
        name: springboot-helloapi
        resources: {}
status: {}
```

Verify that the deployment is up and running according to spec

```
$ k get all -n ckad --show-labels
NAME                             READY   STATUS    RESTARTS   AGE     LABELS
pod/hello-api-64bf75c8db-2qlwm   1/1     Running   0          3m37s   app=hello-api,env=test,pod-template-hash=64bf75c8db
pod/hello-api-64bf75c8db-cvbf6   1/1     Running   0          3m37s   app=hello-api,env=test,pod-template-hash=64bf75c8db
pod/hello-api-64bf75c8db-f8xd2   1/1     Running   0          3m37s   app=hello-api,env=test,pod-template-hash=64bf75c8db

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE     LABELS
deployment.apps/hello-api   3/3     3            3           3m37s   app=hello-api

NAME                                   DESIRED   CURRENT   READY   AGE     LABELS
replicaset.apps/hello-api-64bf75c8db   3         3         3       3m38s   app=hello-api,env=test,pod-template-hash=64bf75c8db
$ 
```

Create a service to expose the deployment

```
$ k expose deployment/hello-api --port=8080 --name hello-api-svc --type=NodePort -n ckad 
service/hello-api-svc exposed

$ minikube service list  | grep hello-api
| ckad                 | hello-api-svc             |         8080 | http://192.168.64.6:32020 |

$ curl http://192.168.64.6:32020/hello/world
Hello world

$ k logs -n ckad -l app=hello-api
2020-10-16 15:09:19,243 INFO  [main] org.springframework.data.repository.config.DeferredRepositoryInitializationListener: Spring Data repositories initialized!
2020-10-16 15:09:19,287 INFO  [main] org.springframework.boot.StartupInfoLogger: Started HelloApiApplication in 27.617 seconds (JVM running for 29.217)
2020-10-16 15:09:19,342 INFO  [task-1] org.hibernate.dialect.Dialect: HHH000400: Using dialect: org.hibernate.dialect.H2Dialect
2020-10-16 15:09:22,935 INFO  [task-1] org.hibernate.engine.transaction.jta.platform.internal.JtaPlatformInitiator: HHH000490: Using JtaPlatform implementation: [org.hibernate.engine.transaction.jta.platform.internal.NoJtaPlatform]
2020-10-16 15:09:22,967 INFO  [task-1] org.springframework.orm.jpa.AbstractEntityManagerFactoryBean: Initialized JPA EntityManagerFactory for persistence unit 'default'
2020-10-16 15:20:04,563 INFO  [http-nio-8080-exec-1] org.apache.juli.logging.DirectJDKLog: Initializing Spring DispatcherServlet 'dispatcherServlet'
2020-10-16 15:20:04,564 INFO  [http-nio-8080-exec-1] org.springframework.web.servlet.FrameworkServlet: Initializing Servlet 'dispatcherServlet'
2020-10-16 15:20:04,585 INFO  [http-nio-8080-exec-1] org.springframework.web.servlet.FrameworkServlet: Completed initialization in 18 ms
2020-10-16 15:20:27,244 INFO  [http-nio-8080-exec-2] com.example.helloApi.web.HelloApi: Hello world
2020-10-16 15:24:54,999 INFO  [http-nio-8080-exec-3] com.example.helloApi.web.HelloApi: Hello world
...
2020-10-16 15:22:08,327 INFO  [http-nio-8080-exec-1] com.example.helloApi.web.HelloApi: Hello world
2020-10-16 15:25:07,565 INFO  [http-nio-8080-exec-2] com.example.helloApi.web.HelloApi: Hello world
2020-10-16 15:25:07,628 INFO  [http-nio-8080-exec-3] com.example.helloApi.web.HelloApi: Hello world
2020-10-16 15:09:27,869 INFO  [main] org.springframework.boot.StartupInfoLogger: Started HelloApiApplication in 31.258 seconds (JVM running for 33.739)
2...
2020-10-16 15:20:22,950 INFO  [http-nio-8080-exec-1] com.example.helloApi.web.HelloApi: Hello Anonymous
2020-10-16 15:20:44,761 INFO  [http-nio-8080-exec-2] com.example.helloApi.web.HelloApi: Hello world
2020-10-16 15:25:07,593 INFO  [http-nio-8080-exec-4] com.example.helloApi.web.HelloApi: Hello world
```
  
</details>  
 
 ## Q6. Update the image in Q5 to `huypuma/springboot-helloapi:0.0.1-MEDUSA` and carry out some simple tests. Verify that server starts failing after a call to `http://<service-ip>:8080/hello/medusa`. Rollback the deployment
 
 <details><summary>Solution</summary>
 
 Validate that there is only 1 entry in `deployment/hello-api`:
 
 ```
 $ k rollout history deployment/hello-api -n ckad
deployment.apps/hello-api 
REVISION  CHANGE-CAUSE
1         <none>
```

Update deployment image

```
$ k set image deployment/hello-api springboot-helloapi=huypuma/springboot-helloapi:0.0.1-MEDUSA -n ckad
deployment.apps/hello-api image updated

$ k rollout history deploy/hello-api -n ckad 
deployment.apps/hello-api 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

Verify that calling `/hello/medusa` triggers a critical server bug which eventually renders all available replicas unserviceable

```
$ curl http://192.168.64.6:32020/hello/medusa
{"timestamp":"2020-10-16T15:53:17.427+00:00","status":500,"error":"Internal Server Error","message":"","path":"/hello/medusa"} 
...
$ curl http://192.168.64.6:32020/hello/world
{"timestamp":"2020-10-16T15:54:07.465+00:00","status":500,"error":"Internal Server Error","message":"","path":"/hello/world"}
```

Rollback the image update to earlier version

```
$ k rollout undo deployment/hello-api -n ckad
deployment.apps/hello-api rolled back

$ k get pod -n ckad --show-labels --watch
NAME                         READY   STATUS              RESTARTS   AGE   LABELS
hello-api-64bf75c8db-5zb84   1/1     Running             0          15s   app=hello-api,env=test,pod-template-hash=64bf75c8db
hello-api-64bf75c8db-8g28q   1/1     Running             0          8s    app=hello-api,env=test,pod-template-hash=64bf75c8db
hello-api-64bf75c8db-gdcw8   0/1     ContainerCreating   0          2s    app=hello-api,env=test,pod-template-hash=64bf75c8db
hello-api-7b577fdfc9-76ggr   1/1     Terminating         0          16m   app=hello-api,env=test,pod-template-hash=7b577fdfc9
hello-api-7b577fdfc9-j4226   1/1     Running             0          16m   app=hello-api,env=test,pod-template-hash=7b577fdfc9
hello-api-7b577fdfc9-76ggr   0/1     Terminating         0          16m   app=hello-api,env=test,pod-template-hash=7b577fdfc9
hello-api-64bf75c8db-gdcw8   1/1     Running             0          8s    app=hello-api,env=test,pod-template-hash=64bf75c8db
hello-api-7b577fdfc9-j4226   1/1     Terminating         0          16m   app=hello-api,env=test,pod-template-hash=7b577fdfc9
hello-api-7b577fdfc9-76ggr   0/1     Terminating         0          16m   app=hello-api,env=test,pod-template-hash=7b577fdfc9
hello-api-7b577fdfc9-76ggr   0/1     Terminating         0          16m   app=hello-api,env=test,pod-template-hash=7b577fdfc9
hello-api-7b577fdfc9-j4226   0/1     Terminating         0          16m   app=hello-api,env=test,pod-template-hash=7b577fdfc9
hello-api-7b577fdfc9-j4226   0/1     Terminating         0          16m   app=hello-api,env=test,pod-template-hash=7b577fdfc9
hello-api-7b577fdfc9-j4226   0/1     Terminating         0          16m   app=hello-api,env=test,pod-template-hash=7b577fdfc9

^C$ k get pod -n ckad --show-labels
NAME                         READY   STATUS    RESTARTS   AGE   LABELS
hello-api-64bf75c8db-5zb84   1/1     Running   0          72s   app=hello-api,env=test,pod-template-hash=64bf75c8db
hello-api-64bf75c8db-8g28q   1/1     Running   0          65s   app=hello-api,env=test,pod-template-hash=64bf75c8db
hello-api-64bf75c8db-gdcw8   1/1     Running   0          59s   app=hello-api,env=test,pod-template-hash=64bf75c8db

```

Verify that the rollback was successful and app responds fine even with erroneous input

```
$ curl http://192.168.64.6:32020/hello/world
Hello world 

$ curl http://192.168.64.6:32020/hello/medusa
Hello medusa
 
$ while [ $i -le 5 ]; do curl http://192.168.64.6:32020/hello/world; i=$(($i+1)); done
Hello worldHello worldHello worldHello worldHello world

```

</details>