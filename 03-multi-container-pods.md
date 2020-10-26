
# Multi-container Pods (10%)

## Objectives
- Understand multi-container pod designs patterns (e.g. ambassador, Adapter, Sidecar)

## References

https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/

## Q1. Create an nginx container in a pod that serve content created by a `alpine/git` init container that clones a git repo specified by environment variable `GIT_REPO` and secret `GIT_TOKEN`. Verify that HTML returned by nginx is really from the container.

<details><summary>Solution</summary>
  
**Prerequisite**: 
- Create a private repo in your github account with an `index.html` with content of your liking
- Create a personal access token (https://github.com/settings/tokens) with appropriate permission. Keep this token SECRET!

**Create config map and secret**

Note: Replace `<value>` below with your own repo and token, employ stronger protection and/or delete the token after this exercise

```
kubectl create configmap website --from-literal=GIT_REPO=<your-own-private-repo> -n ckad
configmap/website created

kubectl create secret generic git-token --from-literal=GIT_TOKEN=<your-own-token> -n ckad
secret/git-token created
```

**Prepare pod's yaml. Add the following:**
- An init container named `git-cloner` that executes command `git clone https://$GIT_TOKEN@$GIT_REPO website`
- A shared volume called `git-volume` that mounts to `/git` in the `git-cloner` container and `/usr/share/nginx/html` in the main `nginx` container

```
$ cat /tmp/pattern-init.yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  volumes:
  - name: git-volume
    emptyDir: {}
  initContainers:
  - name: git-cloner
    image: alpine/git
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: git-volume
      mountPath: /git
    command: ['sh', '-c', 'env; git clone https://$GIT_TOKEN@$GIT_REPO website']
    envFrom:
      - configMapRef:
          name: website
      - secretRef:
          name: git-token
  containers:
  - image: nginx
    name: nginx
    resources: {}
    volumeMounts:
    - name: git-volume
      mountPath: /usr/share/nginx/html
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}


$ k apply -f /tmp/pattern-init.yaml -n ckad
pod/nginx created

$ k get pod -n ckad
NAME    READY   STATUS            RESTARTS   AGE
nginx   0/1     PodInitializing   0          9s

$ k get pod -n ckad
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          30s

```

**Exec into `nginx` container and verify that initialization has completed successfully and nginx is serving the right content**

```
$ k exec nginx -n ckad -c nginx -it -- sh
# cat /usr/share/nginx/html/website/index.html
<html>
  <head>
    <title>Hello HTML</title>
  </head>
  <body>
    <h1>Hello HTML</h1>
    <p>10 Oct 2020 - 15:41</p>
  </body>
</html>

# curl localhost/website/index.html
<html>
  <head>
    <title>Hello HTML</title>
  </head>
  <body>
    <h1>Hello HTML</h1>
    <p>10 Oct 2020 - 15:41</p>
  </body>
</html>
```

</details>

## Q2. Redo Q1 and add an sidecar container that keep the git repo updated. Verify that the HTML returned by nginx is really up to date.

<details><summary>Solution</summary>

Add a sidecar container named `git-updater` that has the same volume mount as `git-cloner` but do `git pull` every minute instead

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  volumes:
  - name: git-volume
    emptyDir: {}
  initContainers:
  - name: git-cloner
    image: alpine/git
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: git-volume
      mountPath: /git
    command: ['sh', '-c', 'env; git clone https://$GIT_TOKEN@$GIT_REPO website']
    envFrom:
      - configMapRef:
          name: website
      - secretRef:
          name: git-token
  containers:
  - image: nginx
    name: nginx
    resources: {}
    volumeMounts:
    - name: git-volume
      mountPath: /usr/share/nginx/html
  - name: git-updater
    image: alpine/git
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: git-volume
      mountPath: /git
    command: ['sh', '-c', 'env; cd website; while true; do sleep 1m; echo "Updating..."; git pull; done']
    envFrom:
      - configMapRef:
          name: website
      - secretRef:
          name: git-token
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}


```

Verify that nginx container still serve data from the right repo

```
# date; curl localhost/website/index.html
Sat Oct 10 14:59:22 UTC 2020
<html>
  <head>
    <title>Hello HTML</title>
  </head>
  <body>
    <h1>Hello HTML</h1>
    <p>10 Oct 2020 - 15:41</p>
  </body>
</html>

```

Update the repo, and check that `git-updated` has update the nginx content to repo's latest

```
# while true; do date; curl localhost/website/index.html; sleep 1m; done;
Sat Oct 10 15:05:33 UTC 2020
<html>
  <head>
    <title>Hello HTML</title>
  </head>
  <body>
    <h1>Hello HTML</h1>
    <p>10 Oct 2020 - 15:41</p>
  </body>
</html>

Sat Oct 10 15:06:33 UTC 2020
<html>
  <head>
    <title>Hello HTML</title>
  </head>
  <body>
    <h1>Hello HTML</h1>
    <p>10 Oct 2020 - updated on 23:05</p>
  </body>
</html>

```

</details>

**TODO**: Add more questions related to Adapter and Ambassador patterns
