# Configuration (18%)

## Objectives
- Understand ConfigMaps
- Understand SecurityContext
- Define an application's resource requirements
- Create and consume Secrets
- Understand ServiceAccounts

## Q1. Create an `alpine/git` pod that clones a git repo specified by environment variable `GIT_REPO` and keep it up-to-date regularly

<details><summary>Solution:</summary>

**Create a ConfigMap to hold configuration**

```
kubectl create configmap website --from-literal=site-name=ckad-prep --from-literal=git_repo=https://github.com/mdn/beginner-html-site-styled.git -n ckad
configmap/website created

kubectl get cm -n ckad
NAME      DATA   AGE
website   2      15s

kubectl describe cm -n ckad
Name:         website
Namespace:    ckad
Labels:       <none>
Annotations:  <none>

Data
====
site-name:
----
ckad-prep
git_repo:
----
https://github.com/mdn/beginner-html-site-styled.git
Events:  <none>
```

**Create the `alpine/git` pod that receives environments values from the `website` ConfigMap**

First, obtain a base template of that pod

```
kubectl run git-pod --image=alpine/git --restart=Never -n ckad -o yaml --dry-run=client > /tmp/git-pod.yaml
```

Then add in environment-related fields and entry command. Tips for recalling ConfigMap wiring syntax
- Search "ConfigMap" in Kubernetes docs
- `kubectl explain pod.spec.containers.env` or `.envFrom`


```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: git-pod
  name: git-pod
spec:
  containers:
  - image: alpine/git
    name: git-pod
    resources: {}
    command: ["sh", "-c", "env; git clone $GIT_REPO website; cd website; while true; do sleep 20; echo $(date)  updating...; git pull; done;"]
    env:
      - name: GIT_REPO
        valueFrom: 
          configMapKeyRef:
            name: website
            key: git_repo
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

```

**Finally create the pod and monitor its logs**

```
kubectl apply -f /tmp/git-pod.yaml -n ckad
pod/git-pod created

kubectl logs -f git-pod -n ckad
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT=443
HOSTNAME=git-pod
SHLVL=1
HOME=/root
GIT_REPO=https://github.com/mdn/beginner-html-site-styled.git
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/git
Cloning into 'website'...

Sun Oct 4 16:27:12 UTC 2020 updating...
Already up to date.
Sun Oct 4 16:27:34 UTC 2020 updating...
Already up to date.
Sun Oct 4 16:27:56 UTC 2020 updating...
Already up to date.

```
</details>

## Q2. From an (existing) environment file, redo Q1

<details><summary>Solution</summary>
  
Assume we have an existing environment file `.env`

```
cat /tmp/.env 
GIT_REPO=https://github.com/mdn/beginner-html-site-styled.git
SITE_NAME=html-basic
```

Create the ConfigMap from this env file

```
$: kubectl create configmap website --from-env-file=/tmp/.env -n ckad
configmap/website created

$: kubectl get cm/website -n ckad -o yaml
apiVersion: v1
data:
  GIT_REPO: https://github.com/mdn/beginner-html-site-styled.git
  SITE_NAME: html-basic
kind: ConfigMap
metadata:
  creationTimestamp: "2020-10-04T16:47:40Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:GIT_REPO: {}
        f:SITE_NAME: {}
    manager: kubectl
    operation: Update
    time: "2020-10-04T16:47:40Z"
  name: website
  namespace: ckad
  resourceVersion: "505586"
  selfLink: /api/v1/namespaces/ckad/configmaps/website
  uid: 50e690f5-963e-4696-bbbb-b9002a76301d
  
```

Create the pod and monitor its log

```
$ cat /tmp/git-pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: git-pod
  name: git-pod
spec:
  containers:
  - image: alpine/git
    name: git-pod
    resources: {}
    command: ["sh", "-c", "env; git clone $GIT_REPO website; cd website; while true; do sleep 20; echo $(date)  updating...; git pull; done;"]
    envFrom:
      - configMapRef:
          name: website
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

$ k apply -f /tmp/git-pod.yaml -n ckad
pod/git-pod created

$ k logs -f git-pod -n ckad
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
HOSTNAME=git-pod
SHLVL=1
HOME=/root
GIT_REPO=https://github.com/mdn/beginner-html-site-styled.git
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
SITE_NAME=html-basic
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/git
Cloning into 'website'...

Sun Oct 4 16:52:59 UTC 2020 updating...
Already up to date.

```

</details>

## Q3. Continue from Q2. Assuming the git repo is private. Create a secret `git-token` to pass the git access token to the pod

<details><summary>Solution</summary>

Create secret `git-token`

```
$ kubectl create secret generic git-secret --from-literal=git-access-token=mock-access-token -n ckad
secret/git-secret created

$ kubectl describe secret git-secret -n ckad
Name:         git-secret
Namespace:    ckad
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
git-access-token:  17 bytes

```

Create the pod with `git-token` secret as environment variables

```
$ cat /tmp/git-pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: git-pod
  name: git-pod
spec:
  containers:
  - image: alpine/git
    name: git-pod
    resources: {}
    command: ["sh", "-c", "env; git clone $GIT_REPO website; cd website; while true; do sleep 20; echo $(date)  updating...; git pull; done;"]
    env:
      - name: GIT_ACCESS_TOKEN
        valueFrom: 
          secretKeyRef:
            name: git-secret
            key: git-access-token
    envFrom:
      - configMapRef:
          name: website
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

$ kubectl apply -f /tmp/git-pod.yaml -n ckad
pod/git-pod created

$ kubectl logs git-pod -n ckad | grep GIT_ACCESS_TOKEN -C1
SITE_NAME=html-basic
GIT_ACCESS_TOKEN=mock-access-token
KUBERNETES_SERVICE_HOST=10.96.0.1

```
</details>  
  

## Q4. Supply an external .properties file with `server.port=8081` to customize listerning port of an spring-boot container running `huypuma/springboot-helloapi`

<details><summary>Solution</summary>
  
Create the `application.properties` file

```
$ echo server.port=8081 > /tmp/application.properties; cat /tmp/application.properties 
server.port=8081
```

Create the ConfigMap with values from the .properties file

*(we'll attempt creating both ConfigMap and Pod in a same yaml file in this exercise)*

```
$ kubectl create configmap appconfig --from-file=/tmp/application.properties -o yaml --dry-run=client > /tmp/hello-api.yaml
$ echo --- >> /tmp/hello-api.yaml 
$ cat /tmp/hello-api.yaml 
apiVersion: v1
data:
  application.properties: |
    server.port=8081
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: appconfig
---

```

Create the pod yaml file and edit it with config map as a volume mount

```
$ kubectl run hello-api --image=huypuma/springboot-helloapi --restart=Never -o yaml --dry-run=client >> /tmp/hello-api.yaml
$ vi /tmp/hello-api.yaml

$ cat /tmp/hello-api.yaml 
apiVersion: v1
data:
  application.properties: |
    server.port=8081
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: appconfig
---
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: hello-api
  name: hello-api
spec:
  volumes:
  - name: config-volume
    configMap:
      name: appconfig
  containers:
  - image: huypuma/springboot-helloapi
    name: hello-api
    resources: {}
    volumeMounts:
    - name: config-volume
      mountPath: /workspace/config
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

```

Create the config map and pod

```
$ kubectl apply -f /tmp/hello-api.yaml -n ckad
configmap/appconfig created
pod/hello-api created
$ k get cm,pod -n ckad
NAME                  DATA   AGE
configmap/appconfig   1      10s

NAME            READY   STATUS    RESTARTS   AGE
pod/alpine      1/1     Running   0          52m
pod/hello-api   1/1     Running   0          10s

$ k logs -f hello-api -n ckad
...(log truncated)...

2020-10-06 01:28:49.491  INFO 1 --- [         task-1] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 5.4.21.Final
2020-10-06 01:28:50.188  INFO 1 --- [         task-1] o.hibernate.annotations.common.Version   : HCANN000001: Hibernate Commons Annotations {5.1.0.Final}
2020-10-06 01:28:50.774  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8081 (http) with context path ''
2020-10-06 01:28:50.782  INFO 1 --- [           main] DeferredRepositoryInitializationListener : Triggering deferred initialization of Spring Data repositories?
2020-10-06 01:28:50.785  INFO 1 --- [           main] DeferredRepositoryInitializationListener : Spring Data repositories initialized!
2020-10-06 01:28:50.840  INFO 1 --- [           main] c.example.helloApi.HelloApiApplication   : Started HelloApiApplication in 8.626 seconds (JVM running for 9.617)
...(log truncated)...

```

Test that it listens to port `8081` as configured in the config map, not the default `8080` we saw previously 

```
$ k get pod -n ckad -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
alpine      1/1     Running   0          57m   172.17.0.8    minikube   <none>           <none>
hello-api   1/1     Running   0          5m    172.17.0.10   minikube   <none>           <none>
```

Within `alpine` pod:

```
/ # curl 172.17.0.10:8080/hello
curl: (7) Failed to connect to 172.17.0.10 port 8080: Connection refused

/ # curl 172.17.0.10:8081/hello
Hello Anonymous
```
</details>  


## Q5. Inside namespace `ckad-test-env`, set a limit of 3 on the number of pods active at any time, request limit of 1.0 on CPU and 1Gi on memory. Validate that creating a deployment with replicate set of 2 succeeds, but scaling to 4 pods partially failed  due to quota restriction. Remove the `count/pods=3` quota restriction and observe that the number of pods is now 4

<details><summary>Solution</summary>

Create the stated quota

```
 kubectl create quota test-env-quota --hard=count/pods=3,requests.cpu=1.0,requests.memory=1Gi -o yaml -n ckad-test-env
 
 $ k describe quota test-env-quota -n ckad-test-env
Name:            test-env-quota
Namespace:       ckad-test-env
Resource         Used  Hard
--------         ----  ----
count/pods       0     3
requests.cpu     0     1
requests.memory  0     1Gi
 
```

Create a deployment with initial 2 replicas having resource's `request.cpu=0.1` and `request.memory=100mb`

```
kubectl create deployment nginx --image=nginx -o yaml --dry-run=client > /tmp/test-env-deploy.yaml

$ cat /tmp/test-env-deploy.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources:
          requests:
            cpu: 0.1
            memory: 200M 
status: {}

$ kubectl apply -f /tmp/test-env-deploy.yaml -n ckad-test-env
deployment.apps/nginx created


$ kubectl get deploy,quota,pod,rs -n ckad-test-env
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   2/2     2            2           20s

NAME                           AGE   REQUEST                                                            LIMIT
resourcequota/test-env-quota   16m   count/pods: 2/3, requests.cpu: 200m/1, requests.memory: 400M/1Gi   

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-b4cdcb7f7-5zdwk   1/1     Running   0          20s
pod/nginx-b4cdcb7f7-l286w   1/1     Running   0          20s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-b4cdcb7f7   2         2         2       20s


$ k describe deploy nginx -n ckad-test-env
...
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
...
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  64s   deployment-controller  Scaled up replica set nginx-b4cdcb7f7 to 2


$ k describe rs -n ckad-test-env
...
Controlled By:  Deployment/nginx
Replicas:       2 current / 2 desired
Pods Status:    2 Running / 0 Waiting / 0 Succeeded / 0 Failed
...
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  84s   replicaset-controller  Created pod: nginx-b4cdcb7f7-5zdwk
  Normal  SuccessfulCreate  84s   replicaset-controller  Created pod: nginx-b4cdcb7f7-l286w
$ 

```

Attempt to scale the deployment to 4 pods, however only 3 pods can be created. Observed the resource limit updated in `test-env-quota` resource

```
$ kubectl scale --replicas=4 deploy nginx -n ckad-test-env
deployment.apps/nginx scaled

$ kubectl get deploy,quota,pod,rs -n ckad-test-env
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   3/4     3            3           4m1s

NAME                           AGE   REQUEST                                                            LIMIT
resourcequota/test-env-quota   19m   count/pods: 3/3, requests.cpu: 300m/1, requests.memory: 600M/1Gi   

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-b4cdcb7f7-5zdwk   1/1     Running   0          4m1s
pod/nginx-b4cdcb7f7-l286w   1/1     Running   0          4m1s
pod/nginx-b4cdcb7f7-lhvvh   1/1     Running   0          27s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-b4cdcb7f7   4         3         3       4m1s


$ kubectl describe deploy -n ckad-test-env
...
Replicas:               4 desired | 3 updated | 3 total | 3 available | 1 unavailable
...


$ kubectl describe rs -n ckad-test-env
...
Replicas:       3 current / 4 desired
Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
...
Events:
  Type     Reason            Age                   From                   Message
  ----     ------            ----                  ----                   -------
  Normal   SuccessfulCreate  10m                   replicaset-controller  Created pod: nginx-b4cdcb7f7-5zdwk
  Normal   SuccessfulCreate  10m                   replicaset-controller  Created pod: nginx-b4cdcb7f7-l286w
  Warning  FailedCreate      6m43s                 replicaset-controller  Error creating: pods "nginx-b4cdcb7f7-gwbm8" is forbidden: exceeded quota: test-env-quota, requested: count/pods=1, used: count/pods=3, limited: count/pods=3
  Warning  FailedCreate      6m43s                 replicaset-controller  Error creating: pods "nginx-b4cdcb7f7-cwv6v" is forbidden: exceeded quota: test-env-quota, requested: count/pods=1, used: count/pods=3, limited: count/pods=3
  Warning  FailedCreate      6m43s                 replicaset-controller  Error creating: pods "nginx-b4cdcb7f7-slxm9" is forbidden: exceeded quota: test-env-quota, requested: count/pods=1, used: count/pods=3, limited: count/pods=3
  Warning  FailedCreate      6m43s                 replicaset-controller  Error creating: pods "nginx-b4cdcb7f7-9qdvz" is forbidden: exceeded quota: test-env-quota, requested: count/pods=1, used: count/pods=3, limited: count/pods=3
  Normal   SuccessfulCreate  6m43s                 replicaset-controller  Created pod: nginx-b4cdcb7f7-lhvvh
  Warning  FailedCreate      6m43s                 replicaset-controller  Error creating: pods "nginx-b4cdcb7f7-kncwg" is forbidden: exceeded quota: test-env-quota, requested: count/pods=1, used: count/pods=3, limited: count/pods=3
  Warning  FailedCreate      6m43s                 replicaset-controller  Error creating: pods "nginx-b4cdcb7f7-4zb7b" is forbidden: exceeded quota: test-env-quota, requested: count/pods=1, used: count/pods=3, limited: count/pods=3
  Warning  FailedCreate      6m43s                 replicaset-controller  Error creating: pods "nginx-b4cdcb7f7-dwgt6" is forbidden: exceeded quota: test-env-quota, requested: count/pods=1, used: count/pods=3, limited: count/pods=3
  Warning  FailedCreate      6m43s                 replicaset-controller  Error creating: pods "nginx-b4cdcb7f7-gg492" is forbidden: exceeded quota: test-env-quota, requested: count/pods=1, used: count/pods=3, limited: count/pods=3
  Warning  FailedCreate      6m42s                 replicaset-controller  Error creating: pods "nginx-b4cdcb7f7-5fhlm" is forbidden: exceeded quota: test-env-quota, requested: count/pods=1, used: count/pods=3, limited: count/pods=3
  Warning  FailedCreate      106s (x8 over 6m41s)  replicaset-controller  (combined from similar events): Error creating: pods "nginx-b4cdcb7f7-rl646" is forbidden: exceeded quota: test-env-quota, requested: count/pods=1, used: count/pods=3, limited: count/pods=3

```

Remove the `count/pods: 3` quota restriction

```
$ kubectl edit quota test-env-quota -n ckad-test-env
resourcequota/test-env-quota edited

$ kubectl get quota -n ckad-test-env
NAME             AGE   REQUEST                                           LIMIT
test-env-quota   34m   requests.cpu: 300m/1, requests.memory: 600M/1Gi   

```

Monitor the namespace for changes to deployment, pods and replica set. It may not be immmediate

```
$ kubectl get deploy,quota,pod,rs -n ckad-test-env
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   4/4     4            4           26m

NAME                           AGE   REQUEST                                           LIMIT
resourcequota/test-env-quota   42m   requests.cpu: 400m/1, requests.memory: 800M/1Gi   

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-b4cdcb7f7-5zdwk   1/1     Running   0          26m
pod/nginx-b4cdcb7f7-l286w   1/1     Running   0          26m
pod/nginx-b4cdcb7f7-lhvvh   1/1     Running   0          23m
pod/nginx-b4cdcb7f7-slvxk   1/1     Running   0          2m5s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-b4cdcb7f7   4         4         4       26m
```

</details>
  

## Q6. Continuing from Q5. Create a job running for 2 minutes that requires minimum 600m CPU and 100M memory. While the job is running, attempt to create another pod with 100m CPU, 100M memory and observed that it will fail. Wait for the job to finish and verify that creating the pod is now possible

<details><summary>Solution</summary>

Create a 2 minutes job with the specified resource requirements
  
```
$ kubectl create job 2m-job --image=busybox -n ckad-test-env -o yaml --dry-run=client -- sleep 2m  > /tmp/test-env-job.yaml

$ vi /tmp/test-env-job.yaml 
$ cat /tmp/test-env-job.yaml 
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: 2m-job
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command:
        - sleep
        - 2m
        image: busybox
        name: 2m-job
        resources:
          requests:
            cpu: 600m
            memory: 100M
      restartPolicy: Never
status: {}

$ kubectl apply -f /tmp/test-env-job.yaml -n ckad-test-env
job.batch/2m-job created

$ kubectl get pod,job -n ckad-test-env
NAME                        READY   STATUS    RESTARTS   AGE
pod/2m-job-8cwmh            1/1     Running   0          19s
pod/nginx-b4cdcb7f7-64966   1/1     Running   0          22h
pod/nginx-b4cdcb7f7-ltpl8   1/1     Running   0          22h
pod/nginx-b4cdcb7f7-mnkrm   1/1     Running   0          22h
pod/nginx-b4cdcb7f7-pwxzg   1/1     Running   0          22h

NAME               COMPLETIONS   DURATION   AGE
job.batch/2m-job   0/1           19s        19s

```

Run another pod with 100m CPU, 100M memory requirements and observed that it failed *(Note: handcraft another pod by hand may take longer than 2 minutes by then the 2m-job would have finished)*

```
$ kubectl run a-pod --image=nginx --requests=cpu=100m,memory=100M -n ckad-test-env
Error from server (Forbidden): pods "a-pod" is forbidden: exceeded quota: test-env-quota, requested: requests.cpu=100m, used: requests.cpu=1, limited: requests.cpu=1
```

Retry after the job has finished

```
$ kubectl run a-pod --image=nginx --requests=cpu=100m,memory=100M -n ckad-test-env
pod/a-pod created


$ k get pod,job,quota -n ckad-test-env
NAME                        READY   STATUS      RESTARTS   AGE
pod/2m-job-g8q66            0/1     Completed   0          3m16s
pod/a-pod                   1/1     Running     0          21s
pod/nginx-b4cdcb7f7-64966   1/1     Running     0          22h
pod/nginx-b4cdcb7f7-ltpl8   1/1     Running     0          22h
pod/nginx-b4cdcb7f7-mnkrm   1/1     Running     0          22h
pod/nginx-b4cdcb7f7-pwxzg   1/1     Running     0          22h

NAME               COMPLETIONS   DURATION   AGE
job.batch/2m-job   1/1           2m7s       3m16s

NAME                           AGE   REQUEST                                           LIMIT
resourcequota/test-env-quota   22h   requests.cpu: 500m/1, requests.memory: 900M/1Gi  

```
</details>

## Q7. Create a service account name `job-account` and run a cron job under that service account

<details><summary>Solution</summary>

Create service account `job-account`

```
$ kubectl create serviceaccount job-account -n ckad-test-env
serviceaccount/job-account created

$ kubectl get sa -n ckad-test-env
NAME          SECRETS   AGE
default       1         22h
job-account   1         9s

$ kubectl describe sa job-account -n ckad-test-env
Name:                job-account
Namespace:           ckad-test-env
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   job-account-token-xxjfg
Tokens:              job-account-token-xxjfg
Events:              <none>

```

Create a cron job running under that service account

```
$ kubectl create cronjob a-cron-job --image=busybox --schedule="*/1 * * * *" -o yaml --dry-run=client -- echo date  > /tmp/test-env-cron.yaml

$ vi /tmp/test-env-cron.yaml 
$ cat /tmp/test-env-cron.yaml 
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  creationTimestamp: null
  name: a-cron-job
spec:
  jobTemplate:
    metadata:
      creationTimestamp: null
      name: a-cron-job
    spec:
      template:
        metadata:
          creationTimestamp: null
          labels:
            job: a-cron
        spec:
          serviceAccountName: job-account
          containers:
          - command:
            - date
            image: busybox
            name: a-cron-job
            resources: 
              requests:
                cpu: 100m
                memory: 10M
          restartPolicy: OnFailure
  schedule: '*/1 * * * *'
status: {}

$ kubectl apply -f /tmp/test-env-cron.yaml -n ckad-test-env
cronjob.batch/a-cron-job created

$ k logs -n ckad-test-env -ljob=a-cron
Fri Oct  9 16:51:08 UTC 2020
Fri Oct  9 16:49:08 UTC 2020
Fri Oct  9 16:50:08 UTC 2020

$ kubectl describe pod -l job=a-cron -n ckad-test-env | grep job-account -C1
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from job-account-token-xxjfg (ro)
Conditions:
--
--
Volumes:
  job-account-token-xxjfg:
    Type:        Secret (a volume populated by a Secret)
--
--
    Type:        Secret (a volume populated by a Secret)
    SecretName:  job-account-token-xxjfg
    Optional:    false
--
--
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from job-account-token-xxjfg (ro)
Conditions:
--
```
  
</details>  


## Q8. Create a pod with a container running image `huypuma/springboot-helloapi` and a container running `alpine` running as UID=2000 and GID=2000. Both shares a volume mapped to `/var/log` in the springboot container and `/var/log/helloapi` in the alpine container. Configure the springboot container to log to `/var/log` using `logging.path` property. Validate that user in alpine container cannot modify logs file generated by the springboot container.

<details><summary>Solution</summary>

Modify solution in Q3 to inject a `logging.path` to the config map. Then add a log volume and the alpine container to the yaml file. The final file should look like the following 

```yaml
$ cat /tmp/hello-api-log.yaml 
apiVersion: v1
data:
  application.properties: |
    server.port=8081
    logging.path=/var/log
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: appconfig
---
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: hello-api
  name: hello-api
spec:
  volumes:
  - name: log-volume
    emptyDir: {}
  - name: config-volume
    configMap:
      name: appconfig
  containers:
  - image: huypuma/springboot-helloapi
    name: hello-api
    resources: {}
    volumeMounts:
    - name: config-volume
      mountPath: /workspace/config
    - name: log-volume
      mountPath: /var/log
  - image: alpine
    name: logs
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/helloapi
    securityContext:
      runAsGroup: 2000
    command: ['sh', '-c', 'sleep 1h']
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Apply the yaml file. Log into the alpine container and verify own UID, GID and those of the log files

```
/var/log/helloapi $ id
uid=2000 gid=2000

/var/log/helloapi $ ls -l
total 4
-rw-r--r--    1 1000     1000          3878 Oct  7 17:13 helloapi-app.log

/var/log/helloapi $ echo "blah" >> helloapi-app.log 
sh: can't create helloapi-app.log: Permission denied
```
</details>  
