# Observability (18%)

## Objectives
- Understand LivenessProbe and ReadinessProbe
- Understand container logging
- Understand how to monitor application in Kubernetes
- Understand debugging in Kubernetes

## Q1. Create a `website-svc` Service that balance traffic to pod having `run=nginx` label. Create an `nginx` container with said label and a readiness probe that passes Ready status only when `/usr/share/nginx/html/website/index.html` is available. Verify that calls to the `website` service always fail because there is no pod available

<details><summary>Solution</summary>

Create the `website-svc` service

```
k create service nodeport website-svc --tcp=80:80 -o yaml -n ckad

$ k get svc -n ckad -o wide
NAME          TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE     SELECTOR
website-svc   NodePort   10.97.37.172   <none>        80:30522/TCP   5m20s   app=website-svc

$ minikube service website-svc -n ckad
|-----------|-------------|-------------|---------------------------|
| NAMESPACE |    NAME     | TARGET PORT |            URL            |
|-----------|-------------|-------------|---------------------------|
| ckad      | website-svc | 80-80/80    | http://192.168.64.6:30522 |
|-----------|-------------|-------------|---------------------------|

```

Create the `nginx` pod with label `app=website-svc`. Skip the readiness probe to verify that earlier service setup is correct

```
$ k run nginx --image=nginx -n ckad -lapp=website-svc
pod/nginx created

$ k get pod,svc -n ckad -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
pod/nginx   1/1     Running   0          18s   172.17.0.5   minikube   <none>           <none>

NAME                  TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE     SELECTOR
service/website-svc   NodePort   10.97.37.172   <none>        80:30522/TCP   8m37s   app=website-svc

$ curl 192.168.64.6:30522
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
```

Recreate the pod, add the stated readiness probe

```
$ cat /tmp/probe-nginx.yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    app: website-svc
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
    readinessProbe:
      exec:
        command: ['ls', '/usr/share/nginx/html/website/index.html']
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}


$ k describe pod nginx -n ckad
Name:         nginx
Namespace:    ckad
...
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  70s               default-scheduler  Successfully assigned ckad/nginx to minikube
  Normal   Pulling    69s               kubelet, minikube  Pulling image "nginx"
  Normal   Pulled     64s               kubelet, minikube  Successfully pulled image "nginx"
  Normal   Created    64s               kubelet, minikube  Created container nginx
  Normal   Started    63s               kubelet, minikube  Started container nginx
  Warning  Unhealthy  5s (x6 over 55s)  kubelet, minikube  Readiness probe failed: ls: cannot access '/usr/share/nginx/html/website/index.html': No such file or directory

```

The `website-svc` is now not accessible

```
$ curl 192.168.64.6:30522
curl: (7) Failed to connect to 192.168.64.6 port 30522: Connection refused
```

</details>


## Q2. Edit the pod and add an `git-cloner` container that clone `https://github.com/mdn/beginner-html-site-styled.git` to `/usr/share/nginx/html/` of `nginx` container. Verify that calls to the `website` service succeed now

<details><summary>Solution</summary>

Edit the pod's yaml file to add a `git-cloner` container. Note: it is important that `git-cloner` is part of `initContainers` and not 

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    app: website-svc
  name: nginx
spec:
  volumes:
  - name: git-volume
    emptyDir: {}
  containers:
  - image: nginx
    name: nginx
    resources: {}
    volumeMounts:
    - name: git-volume
      mountPath: /usr/share/nginx/html
    readinessProbe:
      exec:
        command: ['ls', '/usr/share/nginx/html/website/index.html']
  initContainers:
  - image: alpine/git
    name: git-cloner
    command: ['git', 'clone', 'https://github.com/mdn/beginner-html-site-styled.git', 'website']
    volumeMounts:
    - name: git-volume
      mountPath: /git
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Verify that the pod has now gone thru init, probed for readiness to ready

```
$ k apply -f /tmp/probe-nginx.yaml -n ckad
pod/nginx created

$ k get pod -n ckad
NAME    READY   STATUS     RESTARTS   AGE
nginx   0/1     Init:0/1   0          3s

$ k get pod -n ckad
NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Running   0          19s

$ k get pod -n ckad
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          22s

$ curl 192.168.64.6:30522/website/index.html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>My test page</title>
    <link href="http://fonts.googleapis.com/css?family=Open+Sans" rel="stylesheet" type="text/css">
    <link href="styles/style.css" rel="stylesheet" type="text/css">
  </head>
  <body>
    <h1>Mozilla is cool</h1>
    <img src="images/firefox-icon.png" alt="The Firefox logo: a flaming fox surrounding the Earth.">

    <p>At Mozilla, we’re a global community of</p>

    <ul> <!-- changed to list in the tutorial -->
      <li>technologists</li>
      <li>thinkers</li>
      <li>builders</li>
    </ul>

    <p>working together to keep the Internet alive and accessible, so people worldwide can be informed contributors and creators of the Web. We believe this act of human collaboration across an open platform is essential to individual growth and our collective future.</p>

    <p>Read the <a href="https://www.mozilla.org/en-US/about/manifesto/">Mozilla Manifesto</a> to learn even more about the values and principles that guide the pursuit of our mission.</p>
  </body>
</html>
```
  
</details>  

## Q3. Create a `hello-everyone` Service that balance traffic to pod having `app=hello-everyone` label. However, create a container with said label with image `huypuma/springboot-helloapi:0.0.1-MEDUSA`. Verify that calls to that service starts failling once the name `medusa` is passed in.

<details><summary>Solution</summary>

Create the `hello-everyone` service

```
$ k create svc nodeport hello-everyone --tcp=8080 -n ckad
service/hello-everyone created

$ k describe service hello-everyone -n ckad
Name:                     hello-everyone
Namespace:                ckad
Labels:                   app=hello-everyone
Annotations:              <none>
Selector:                 app=hello-everyone
Type:                     NodePort
IP:                       10.97.185.220
Port:                     8080  8080/TCP
TargetPort:               8080/TCP
NodePort:                 8080  30701/TCP
Endpoints:                <none>
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Create the pod

```
$ k run hello-medusa --image=huypuma/springboot-helloapi:0.0.1-MEDUSA -lapp=hello-everyone -n ckad
pod/hello-medusa created

$ minikube service list | grep hello-everyone
| ckad                 | hello-everyone            | 8080/8080    | http://192.168.64.6:30701 |

$ curl http://192.168.64.6:30701/hello/someone
Hello someone

$ curl http://192.168.64.6:30701/hello/another
Hello another

$ curl http://192.168.64.6:30701/hello/medusa
{"timestamp":"2020-10-11T16:52:25.793+00:00","status":500,"error":"Internal Server Error","message":"","path":"/hello/medusa"}


$ curl http://192.168.64.6:30701/hello/another
{"timestamp":"2020-10-11T16:52:41.764+00:00","status":500,"error":"Internal Server Error","message":"","path":"/hello/another"}

$ curl http://192.168.64.6:30701/hello/someone
{"timestamp":"2020-10-11T16:52:50.854+00:00","status":500,"error":"Internal Server Error","message":"","path":"/hello/someone"} 
```

</details>

## Q4. Edit the pod and add a liveness probe that check for http call to `/hello/world` is successful. Verify that service can eventually recover even after the name `medusa` is passed in.

<details><summary>Solution</summary>

Add a liveness probe that query `/hello/world`

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    app: hello-everyone
  name: hello-medusa
spec:
  containers:
  - image: huypuma/springboot-helloapi:0.0.1-MEDUSA
    name: hello-medusa
    resources: {}
    livenessProbe:
      httpGet:
        path: /hello/world
        port: 8080
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Re-create the pod and verify that once liveness probe detects issue with the container, it will attempt a restart and recover the application state.

```
$ curl http://192.168.64.6:30701/hello/someone
Hello someone

$ curl http://192.168.64.6:30701/hello/medusa
{"timestamp":"2020-10-11T17:01:20.337+00:00","status":500,"error":"Internal Server Error","message":"","path":"/hello/medusa"}

$ curl http://192.168.64.6:30701/hello/someone
{"timestamp":"2020-10-11T17:01:24.309+00:00","status":500,"error":"Internal Server Error","message":"","path":"/hello/someone"}

$ k get pod -n ckad
NAME           READY   STATUS    RESTARTS   AGE
hello-medusa   1/1     Running   1          87s

$ curl http://192.168.64.6:30701/hello/someone
Hello someone 

```

Verify that liveness probe is responsible for the restart

```
$ k describe pod hello-medusa -n ckad
Name:         hello-medusa
Namespace:    ckad
...
    State:          Running
      Started:      Mon, 12 Oct 2020 01:01:41 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    143
      Started:      Mon, 12 Oct 2020 01:00:32 +0800
      Finished:     Mon, 12 Oct 2020 01:01:41 +0800
    Ready:          True
    Restart Count:  1
    Liveness:       http-get http://:8080/hello/world delay=0s timeout=1s period=10s #success=1 #failure=3
...
Events:
...
  Warning  Unhealthy  2m58s (x3 over 3m18s)  kubelet, minikube  Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    2m58s                  kubelet, minikube  Container hello-medusa failed liveness probe, will be restarted
  Normal   Pulled     2m57s (x2 over 4m6s)   kubelet, minikube  Container image "huypuma/springboot-helloapi:0.0.1-MEDUSA" already present on machine
  Normal   Created    2m57s (x2 over 4m6s)   kubelet, minikube  Created container hello-medusa
  Normal   Started    2m57s (x2 over 4m6s)   kubelet, minikube  Started container hello-medusa
  Warning  Unhealthy  2m48s (x2 over 3m58s)  kubelet, minikube  Liveness probe failed: Get http://172.17.0.5:8080/hello/world: dial tcp 172.17.0.5:8080: connect: connection refused

```

</details>