# Services & Networking (13%)

## Objectives
- Understand Services
- Demonstrate basic understanding of NetworkPolicies

## Q1. Create a deployment of 3 replicas of image `huypuma/springboot-helloapi` with label `app: helloapi`, `track: stable`. Create another deployment called `release` with image `huypuma/springboot-helloapi:0.0.1-MEDUSA` with 1 replica and label `app: helloapi`, `track: canary`. Create a service listerning at port 80 and exposing all pods with label `app: helloapi`. Simulate about 1000 calls to `/hello/<name>/world` to that services. Monitor how the service balance traffic to its backing pods

<details><summary>Solution</summary>
  
Create deployment `helloapi-deploy` with the following yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: helloapi-deploy
  name: helloapi-deploy
  namespace: ckad
spec:
  replicas: 3
  selector:
    matchLabels:
      app: helloapi
      track: stable
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: helloapi
        track: stable
    spec:
      containers:
      - image: huypuma/springboot-helloapi
        name: springboot-helloapi
        resources: {}
status: {}

```

Create another deployment `release` with image `huypuma/springboot-helloapi:0.0.1-MEDUSA`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: helloapi-release
  name: helloapi-release
  namespace: ckad
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloapi
      track: canary
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: helloapi
        track: canary
    spec:
      containers:
      - image: huypuma/springboot-helloapi:0.0.1-MEDUSA
        name: springboot-helloapi
        resources: {}
status: {}

```

Create a service serving incoming requests at port 80 and exposing all pods with `app: helloapi` labels . Hint: use `k create service nodeport helloapi-svc --tcp=80:8080 -o yaml --dry-run=client > helloapi-svc.yaml` to create the base template

```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: helloapi-svc
  name: helloapi-svc
spec:
  ports:
  - name: 80-8080
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: helloapi
  type: NodePort
status:
  loadBalancer: {}
```



Simulate a 100 calls to `helloapi-svc` and monitor traffic to backend pods

```
$ minikube service list | grep helloapi
| ckad                 | helloapi-svc              | 80-8080/80   | http://192.168.64.6:30994 |

$ while [ $i -lt 2000 ]; do curl http://192.168.64.6:30994/hello/world-$i; i=$(($i+1)); done
Hello world-0Hello world-1Hello world-2Hello world-3Hello world-4Hello world-5Hello world-6Hello world-7Hello world-8Hello world-9Hello world-10Hello world-11Hello world-12Hello world-13Hello world-14Hello world-15Hello world-16Hello world-17Hello world-18Hello world-19Hello world-20Hello world-21Hello world-22Hello world-23Hello world-24Hello world-25Hello world-26Hello world-27Hello world-28Hello world-29Hello world-30Hello world-31Hello world-32Hello world-33Hello world-34Hello world-35Hello world-36Hello world-37Hello world-38Hello world-39Hello world-40Hello world-41Hello world-42Hello world-43Hello world-44Hello world-45Hello world-46Hello world-47Hello world-48Hello world-49Hello world-50Hello world-51Hello world-52Hello world-53Hello world-54Hello world-55Hello world-56Hello world-57Hello world-58Hello world-59Hello world-60Hello world-61Hello world-62Hello world-63Hello world-64Hello world-65Hello world-66Hello world-67Hello world-68Hello world-69Hello world-70Hello world-71Hello world-72Hello world-73Hello world-74Hello world-75Hello world-76Hello world-77Hello world-78Hello world-79Hello world-80Hello world-81Hello world-82Hello world-83Hello world-84Hello world-85Hello world-86Hello world-87Hello world-88Hello world-89Hello world-90Hello world-91Hello world-92Hello world-93Hello world-94Hello world-95Hello world-96Hello world-97Hello world-98Hello world-99...


$ for podname in `k get pod -n ckad -o template='{{range .items}}{{.metadata.name}}{{printf "\n"}}{{end}}'`; do echo "Pod $podname:"; kubectl logs $podname -n ckad | grep "Hello world" | wc; done
Pod helloapi-deploy-844f88d9f9-b2dx2:
     507    4056   77027
Pod helloapi-deploy-844f88d9f9-v27m8:
     551    4408   83718
Pod helloapi-deploy-844f88d9f9-w6cn2:
     496    3968   75365
Pod helloapi-release-68b9bfdddf-x9nfd:
     546    4368   82968

```
</details>

## Q2. Update `helloapi-svc` to direct traffic to only pod with `track: stable` branch. Monitor traffic and report result.

<details><summary>Solution</summary>

Update `helloapi-svc` yaml file and apply the change

```
$ cat helloapi-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: helloapi-svc
  name: helloapi-svc
spec:
  ports:
  - name: 80-8080
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: helloapi
    track: stable
  type: NodePort
status:
  loadBalancer: {}
  
$ k apply -f helloapi-svc.yaml -n ckad
service/helloapi-svc configured

$ k describe svc helloapi-svc -n ckad
Name:                     helloapi-svc
Namespace:                ckad
Labels:                   app=helloapi-svc
Annotations:              Selector:  app=helloapi,track=stable
Type:                     NodePort
IP:                       10.109.66.239
Port:                     80-8080  80/TCP
TargetPort:               8080/TCP
NodePort:                 80-8080  30994/TCP
Endpoints:                172.17.0.4:8080,172.17.0.5:8080,172.17.0.7:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Simulate 1000 calls and report results

```
$ while [ $i -lt 1000 ]; do curl http://192.168.64.6:30994/hello/world-$i; i=$(($i+1)); done
Hello world-0Hello world-1Hello world-2Hello world-3Hello world-4Hello world-5Hello world-6Hello world-7Hello world-8Hello world-9Hello world-10Hello world-11Hello world-12Hello world-13Hello world-14Hello world-15Hello world-16Hello world-17Hello world-18Hello world-19Hello world-20Hello world-21Hello world-22Hello world-23Hello world-24Hello world-25Hello world-26Hello world-27Hello world-28Hello world-29Hello world-30Hello world-31Hello world-32Hello world-33Hello world-34Hello world-35Hello world-36Hello world-37Hello world-38Hello world-39Hello world-40Hello world-41Hello world-42Hello world-43Hello world-44Hello world-45Hello world-46Hello world-47Hello world-48Hello world-49Hello world-50Hello world-51Hello world-52Hello...

$ for podname in `k get pod -n ckad -o template='{{range .items}}{{.metadata.name}}{{printf "\n"}}{{end}}'`; do echo "Pod $podname:"; kubectl logs $podname -n ckad | grep "Hello world" | wc; done
Pod helloapi-deploy-844f88d9f9-b2dx2:
     864    6912  131290
Pod helloapi-deploy-844f88d9f9-v27m8:
     873    6984  132665
Pod helloapi-deploy-844f88d9f9-w6cn2:
     817    6536  124146
Pod helloapi-release-68b9bfdddf-x9nfd:
     546    4368   82968

```
  
</details>


## Q3. Create a `frontend` pod running nginx, a `database` pod running postgres. Create a default deny network policy but allowing traffic from `frontend` to `app: helloapi` pods and `app: helloapi` to `database`. 

Note: If you are using minikube for preparation, you'll need to specify a CNI network pluggin (e.g. Cilium) during minikube start up. And you may have to use an OS other than MacOS. I'm using Cilum on Ubuntu following instruction at https://docs.cilium.io/en/v1.8/gettingstarted/minikube/

<details><summary>Solution</summary>
  
Creating `frontend` and `database` pods

```
$ k run frontend --image=nginx -l app=frontend -n ckad
pod/frontend created

$ k run database --image=postgres --env=POSTGRES_PASSWORD=secret -n ckad
pod/database created

```

Verify that `frontend` can call `app: helloapi` pods and `helloapi-svc` service

```
$ k get pod,netpol,svc -n ckad -o wide
NAME                                    READY   STATUS    RESTARTS   AGE    IP           NODE       NOMINATED NODE   READINESS GATES
pod/database                            1/1     Running   1          28m    10.88.0.9    minikube   <none>           <none>
pod/frontend                            1/1     Running   1          33m    10.88.0.4    minikube   <none>           <none>
pod/helloapi-deploy-844f88d9f9-b2dx2    1/1     Running   2          116m   10.88.0.5    minikube   <none>           <none>
pod/helloapi-deploy-844f88d9f9-v27m8    1/1     Running   2          116m   10.88.0.8    minikube   <none>           <none>
pod/helloapi-deploy-844f88d9f9-w6cn2    1/1     Running   2          116m   10.88.0.6    minikube   <none>           <none>
pod/helloapi-release-68b9bfdddf-x9nfd   1/1     Running   1          106m   10.88.0.10   minikube   <none>           <none>

NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service/helloapi-svc   NodePort   10.109.66.239   <none>        80:30994/TCP   95m   app=helloapi,track=stable


$ k exec frontend -n ckad -it -- sh

# curl http://10.109.66.239/hello/frontend   # curl via helloapi-svc
Hello frontend# 

# curl http://10.88.0.6:8080/hello/frontend  # curl directly to a pod
Hello frontend# 
 
```

Create a default-deny network policy

```
$ cat netpol.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

$ k apply -f netpol.yaml -n ckad
networkpolicy.networking.k8s.io/default-deny-all created

root@frontend:/# curl 10.96.39.196/hello/frontend         # curl via helloapi-svc whose ip has changed at this point
curl: (7) Failed to connect to 10.96.39.196 port 80: Connection timed out

root@frontend:/# curl 10.0.0.40/hello/frontend            # curl directly to a pod
curl: (7) Failed to connect to 10.0.0.40 port 80: Connection timed out

```

Create NetworkPolicy allowing traffic front `frontend` to `app: helloapi` and `app:helloapi` to `database`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-helloapi-database 
spec:
  podSelector:
    matchLabels:
      app: helloapi	 
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
  egress:
  - to:
    - podSelector:
        matchLabels:
          run: database
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend 
spec:
  podSelector:
    matchLabels:
      app: frontend	 
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: helloapi
  policyTypes:
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database 
spec:
  podSelector:
    matchLabels:
      run: database	 
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: helloapi
  policyTypes:
  - Ingress

```

Verify that traffic is allowed between `frontend` and `app: helloapi`

```
root@frontend:/# curl 10.96.39.196/hello/frontend
Hello frontend

root@frontend:/# curl 10.0.0.40/hello/frontend
curl: (7) Failed to connect to 10.0.0.40 port 80: Connection refused
root@frontend:/# curl 10.0.0.40:8080/hello/frontend
Hello frontend

```

Verify that elsewhere traffic is still denied

```
root@nginx1:/# curl 10.0.0.197 
curl: (7) Failed to connect to 10.0.0.197 port 80: Connection timed out
root@nginx1:/# curl 10.0.0.40  
curl: (7) Failed to connect to 10.0.0.40 port 80: Connection timed out
root@nginx1:/# 
```

</details>
