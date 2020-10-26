# Core Concepts (13%)

## Objectives
- Understand basic Kubernetes API Primities
- Create and configure basic pods

## Q1. Find all resource types supported by Kubernetes

<details><summary>Solution:</summary>

Use command `kubectl api-resources` to list all supported resource types and their short names if available

```
kubectl api-resources
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
bindings                                                                      true         Binding
componentstatuses                 cs                                          false        ComponentStatus
configmaps                        cm                                          true         ConfigMap
endpoints                         ep                                          true         Endpoints
...
```
</p>
</details>

## Q2. List all pods, services and deployments in namespace `ckad`

<details><summary>Solution:</summary>

**Option 1**: Use `kubectl get all`

```
kubectl get all -n ckad

NAME                                 READY   STATUS              RESTARTS   AGE
pod/my-deployment-5c75bd4b85-xh7wr   0/1     ContainerCreating   0          5s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-deployment   0/1     1            0           5s

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/my-deployment-5c75bd4b85   1         1         0       5s
```

**Option 2**: Specify multiple resource names separated by commas (,)

```
kubectl get pods,services,deployments -n ckad
NAME                                 READY   STATUS             RESTARTS   AGE
pod/my-deployment-5c75bd4b85-xh7wr   0/1     CrashLoopBackOff   4          2m12s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-deployment   0/1     1            0           2m12s
```

</details>

## Q3. Run an nginx pod named `nginx-server` in `ckad` namespace

<details><summary>Solution:</summary>

```
kubectl run nginx-server --image=nginx -n ckad
pod/nginx-server created

kubectl get pod -n ckad
NAME                             READY   STATUS             RESTARTS   AGE
nginx-server                     1/1     Running            0          22s

kubectl describe pod nginx-server -n ckad
Name:         nginx-server
Namespace:    ckad
Priority:     0
Node:         minikube/192.168.64.6
Start Time:   Sun, 04 Oct 2020 21:12:13 +0800
Labels:       run=nginx-server
Annotations:  <none>
Status:       Running
IP:           172.17.0.9
IPs:
  IP:  172.17.0.9
Containers:
  nginx-server:
    Container ID:   docker://bd8b66044936467efde1b1dfae3778dc2ff89de996ae4823a3275f6df6115a18
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:9a1f8ed9e2273e8b3bbcd2e200024adac624c2e5c9b1d420988809f5c0c41a5e
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 04 Oct 2020 21:12:22 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-6nm2l (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-6nm2l:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-6nm2l
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  62s   default-scheduler  Successfully assigned ckad/nginx-server to minikube
  Normal  Pulling    61s   kubelet, minikube  Pulling image "nginx"
  Normal  Pulled     53s   kubelet, minikube  Successfully pulled image "nginx"
  Normal  Created    53s   kubelet, minikube  Created container nginx-server
  Normal  Started    53s   kubelet, minikube  Started container nginx-server
```

</details>

## Q4. Monitor log output of `nginx-server` pod created in Q3

<details><summary>Solution:</summary>
  
```
k logs nginx-server -n ckad -f
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
172.17.0.10 - - [04/Oct/2020:13:21:23 +0000] "GET / HTTP/1.1" 200 612 "-" "Wget" "-"
```

</details>

## Q5. Obtain IP address of `nginx-server` pod

<details><summary>Solution:</summary>

**Option 1**: use `-o wide` option when listing pods

```
kubectl get pods -n ckad -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
nginx-server   1/1     Running   0          15m   172.17.0.9   minikube   <none>           <none>
```

**Option 2**: Use `kubectl describe`

```
kubectl describe pod nginx-server -n ckad | grep IP
IP:           172.17.0.9
IPs:
  IP:  172.17.0.9
```

**Option 3**: Use output's Go-template or jsonpath

```
NGINX_SERVER_IP=$(k get pod nginx-server -n ckad -o template='{{.status.podIP}}'); echo $NGINX_SERVER_IP
172.17.0.9

k get pod nginx-server -n ckad -o jsonpath='{.status.podIP}'
172.17.0.9
```
</details>

## Q6. Create another pod to send a test http request to `nginx-server` pod

<details><summary>Solution:</summary>
  
```
kubectl run tmp-pod --image=busybox -n ckad --restart=Never -it --image-pull-policy=IfNotPresent -- sh
If you don't see a command prompt, try pressing enter.
/ # wget -O - 172.17.0.9
Connecting to 172.17.0.9 (172.17.0.9:80)
writing to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
-                    100% |*************************************************************************************************************************************|   612  0:00:00 ETA
written to stdout
/ # exit
```

If you still have the log monitoring created in Q4 running, you will notice an extra line has appeared

```
kubectl logs nginx-server -n ckad -f
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
172.17.0.10 - - [04/Oct/2020:13:21:23 +0000] "GET / HTTP/1.1" 200 612 "-" "Wget" "-"


172.17.0.8 - - [04/Oct/2020:13:43:39 +0000] "GET / HTTP/1.1" 200 612 "-" "Wget" "-"
```

</details>

## Q7. Delete the temporary pod created in Q6

<details><summary>Solution:</summary>
  
```
kubectl get pods -n ckad
NAME           READY   STATUS      RESTARTS   AGE
nginx-server   1/1     Running     0          35m
tmp-pod        0/1     Completed   0          4m21s


kubectl delete po tmp-pod -n ckad --force
warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "tmp-pod" force deleted
```
</details>

## Q8. Replace Q6 and Q7 with a command that create a temporary pod

<details><summary>Solution:</summary>
  
```
kubectl run tmp-pod --image=busybox --rm -n ckad -it --image-pull-policy=IfNotPresent -- sh
If you don't see a command prompt, try pressing enter.
/ # wget -O - 172.17.0.9
Connecting to 172.17.0.9 (172.17.0.9:80)
writing to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
-                    100% |*************************************************************************************************************************************|   612  0:00:00 ETA
written to stdout
/ # exit
Session ended, resume using 'kubectl attach tmp-pod -c tmp-pod -i -t' command when the pod is running
pod "tmp-pod" deleted
```

</details>