# Deploying Traefik as Ingress Controller for Your Kubernetes Cluster

Traefik is an open-source Edge Router that makes publishing your services a fun and easy experience. It receives requests on behalf of your system and finds out which components are responsible for handling them.

This guide explains how to use Traefik as an Ingress controller for a Kubernetes cluster.

### Prerequisites

A working Kubernetes cluster.

_To download below yaml files you can clone the git url:_ 
```
# git clone https://github.com/ajinkyaingole30/traefik.git
```

### Step 1: Role Based Access Control configuration
Kubernetes introduces Role Based Access Control (RBAC) in 1.6+ to allow fine-grained control of Kubernetes resources and API.

If your cluster is configured with RBAC, you will need to authorize Traefik to use the Kubernetes API. There are two ways to set up the proper permission: Via namespace-specific RoleBindings or a single, global ClusterRoleBinding.

RoleBindings per namespace enable to restrict granted permissions to the very namespaces only that Traefik is watching over, thereby following the least-privileges principle. This is the preferred approach if Traefik is not supposed to watch all namespaces, and the set of namespaces does not change dynamically. Otherwise, a single ClusterRoleBinding must be employed.

For the sake of simplicity, this guide will use a ClusterRoleBinding:
```
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
    - extensions
    resources:
    - ingresses/status
    verbs:
    - update
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
  
  
# kubectl create -f traefik-rbac.yaml
```

### Step 2: Deploy Traefik using a Deployment or DaemonSet
It is possible to use Traefik with a Deployment or a DaemonSet object, whereas both options have their own pros and cons:

- The scalability can be much better when using a Deployment, because you will have a Single-Pod-per-Node model when using a DaemonSet, whereas you may need less replicas based on your environment when using a Deployment.

- DaemonSets automatically scale to new nodes, when the nodes join the cluster, whereas Deployment pods are only scheduled on new nodes if required.

- DaemonSets ensure that only one replica of pods run on any single node. Deployments require affinity settings if you want to ensure that two pods don't end up on the same node.

- DaemonSets can be run with the NET_BIND_SERVICE capability, which will allow it to bind to port 80/443/etc on each host. This will allow bypassing the kube-proxy, and reduce traffic hops. Note that this is against the Kubernetes Best Practices Guidelines, and raises the potential for scheduling/scaling issues. Despite potential issues, this remains the choice for most ingress controllers.

- If you are unsure which to choose, start with the Daemonset.

Here we used daemonset
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
      name: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik:v1.7
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8080
          hostPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO


# kubectl create -f trefik-daemonset.yaml
```

### Step 3: Create NodePorts for External Access
```
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
  type: NodePort
  
# kubectl create -f traefik-svc.yaml
```
To check status of pods
```
# kubectl get pods --all -n kube-system | grep traefik
```
Now, let’s verify that the Service was created:
```
# kubectl describe svc traefik-ingress-service --namespace=kube-system

Name:                     traefik-ingress-service

Namespace:                kube-system

Labels:                   <none>

Annotations:              <none>

Selector:                 k8s-app=traefik-ingress-lb

Type:                     NodePort

IP:                       10.102.215.64

Port:                     web  80/TCP

TargetPort:               80/TCP

NodePort:                 web  30565/TCP

Endpoints:                172.17.0.6:80

Port:                     admin  8080/TCP

TargetPort:               8080/TCP

NodePort:                 admin  30729/TCP

Endpoints:                172.17.0.6:8080

Session Affinity:         None

External Traffic Policy:  Cluster

Events:                   <none>
```  
As you see, now we have two NodePorts (“web” and “admin”) that route to the 80 and 8080 container ports of the Traefik Ingress controller. The “admin” NodePort will be used to access the Traefik Web UI and the “web” NodePort will be used to access services exposed via Ingress.

### Step 4: Accessing Traefik

To access the Traefik Web UI in the browser, you can use the “admin” NodePort 30729 (please note that your NodePort value might differ)  

Access traefik dashboard on browser http://localhost:30729

### Step 5: Adding Ingress to the Cluster

Now we have Traefik as the Ingress Controller in the Kubernetes cluster. However, we still need to define the Ingress resource and a Service that exposes Traefik Web UI.

Let’s first create a Service:
```
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - name: web
    port: 80
    targetPort: 8080
    
    
# kubectl create -f traefik-webui-svc.yaml
```
Let’s verify that the Service was created:

```
# kubectl describe svc traefik-web-ui --namespace=kube-system

Name:              traefik-web-ui

Namespace:         kube-system

Labels:            <none>

Annotations:       <none>

Selector:          k8s-app=traefik-ingress-lb

Type:              ClusterIP

IP:                10.98.230.58

Port:              web  80/TCP

TargetPort:        8080/TCP

Endpoints:         172.17.0.6:8080

Session Affinity:  None

Events:            <none>
```
Next, we need to create an Ingress resource pointing to the Traefik Web UI backend.
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  rules:
  - host: traefik-ui.minikube
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-web-ui
          servicePort: web
          
          
# kubectl create -f traefik-ingress.yaml  
```
You should now be able to see Traefik dashboard http://localhost:<admin_NodePort>
  
### Step 6: Implementing Name-Based Routing
Now, let’s demonstrate how Traefik Ingress Controller can be used to set up name-based routing for a list of frontends. We will create three Deployments with simple single-page websites displaying images of animals: bear, hare, and moose.

Each Deployment will have two Pod replicas, and each Pod will serve the “animal” websites on the containerPort 80.
```
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: bear
  labels:
    app: animals
    animal: bear
spec:
  replicas: 2
  selector:
    matchLabels:
      app: animals
      task: bear
  template:
    metadata:
      labels:
        app: animals
        task: bear
        version: v0.0.1
    spec:
      containers:
      - name: bear
        image: supergiantkir/animals:bear
        ports:
        - containerPort: 80
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: moose
  labels:
    app: animals
    animal: moose
spec:
  replicas: 2
  selector:
    matchLabels:
      app: animals
      task: moose
  template:
    metadata:
      labels:
        app: animals
        task: moose
        version: v0.0.1
    spec:
      containers:
      - name: moose
        image: supergiantkir/animals:moose
        ports:
        - containerPort: 80
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: hare
  labels:
    app: animals
    animal: hare
spec:
  replicas: 2
  selector:
    matchLabels:
      app: animals
      task: hare
  template:
    metadata:
      labels:
        app: animals
        task: hare
        version: v0.0.1
    spec:
      containers:
      - name: hare
        image: supergiantkir/animals:hare
        ports:
        - containerPort: 80
        
        
# kubectl create -f animals-deployment.yaml
```
Now, let’s create a Service for each Deployment to make the Pods accessible:
```

---
apiVersion: v1
kind: Service
metadata:
  name: bear
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: animals
    task: bear
---
apiVersion: v1
kind: Service
metadata:
  name: moose
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: animals
    task: moose
---
apiVersion: v1
kind: Service
metadata:
  name: hare
  annotations:
    traefik.backend.circuitbreaker: "NetworkErrorRatio() > 0.5"
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: animals
    task: hare
    
    
# kubectl create -f animals-svc.yaml
```
Finally, let’s create an Ingress with three frontend-backend pairs for each Deployment. bear.animal.com , moose.animal.com , and hare.animal.come will be our frontends pointing to corresponding backend Services.
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: animals
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: hare.animal.com
    http:
      paths:
      - path: /
        backend:
          serviceName: hare
          servicePort: http
  - host: bear.animal.com
    http:
      paths:
      - path: /
        backend:
          serviceName: bear
          servicePort: http
  - host: moose.animal.com
    http:
      paths:
      - path: /
        backend:
          serviceName: moose
          servicePort: http
          
          
# kubectl create -f animals-ingress.yaml
```
Now, inside the Traefik dashboard and you should see a frontend for each host along with a list of corresponding backends.

If you edit your /etc/hosts you should be able to access the animal websites in your browser.
```
# vi /etc/hosts

10.32.0.3       bear.animal.com
10.32.0.4       bear.animal.com
10.40.0.6       hare.animal.com
10.32.0.5       hare.animal.com
10.40.0.5       moose.animal.com
10.40.0.4       moose.animal.com




# curl bear.example.com

<!DOCTYPE html>
<html>
<head>
        <title>Bear</title>
<style>
body {background-image: url("img/bear.jpg");}

</style>
</head>
<body>

</body>
</html>
```
Access Traefik dashboard on browser at http://localhost:<admin_NodePort> 

To access specific website on browser you should use the “web” NodePort. For example,
```
http://bare.animal.com<web_NodePort>
http://hare.animal.com<web_NodePort>
http://moose.animal.com<web_NodePort>
```
We can also reconfigure three frontends to serve under one domain like this:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: all-animals
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - host: animals.com
    http:
      paths:
      - path: /bear
        backend:
          serviceName: bear
          servicePort: http
      - path: /moose
        backend:
          serviceName: moose
          servicePort: http
      - path: /hare
        backend:
          serviceName: hare
          servicePort: http
```          
If you activate this Ingress, all three animals will be accessible under one domain — animals.minikube — using corresponding paths. Don’t forget to add this domain to /etc/hosts .
```
# vi /etc/hosts

127.0.0.1 animals.com
```
#### To access service
```
http://animals.com:<web_NodePort>/moose/ 
```
### Conclusion

Ingress is a powerful tool for routing external traffic to corresponding backend services in your Kubernetes cluster. Users can implement Ingress using a number of Ingress controllers supported by Kubernetes. Here we used Traefik Ingress controller that supports name-based routing, load balancing, and other common tasks of Ingress controllers. 


