# traefik

#### Step 1: Enabling RBAC

kubectl create -f traefik-rbac.yaml

#### Step 2: Deploy Traefik to a Cluster

kubectl create -f trefik-daemonset.yaml

#### Step 3: Create NodePorts for External Access

kubectl create -f traefik-svc.yaml

kubectl get pods --all -n kube-system | grep traefik

##### To verify the service was created

kubectl describe svc traefik-ingress-service --namespace=kube-system

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
  
##### As you see, now we have two NodePorts (“web” and “admin”) that route to the 80 and 8080 container ports of the Traefik Ingress controller. The “admin” NodePort will be used to access the Traefik Web UI and the “web” NodePort will be used to access services exposed via Ingress.

#### Step 4: Accessing Traefik

##### To access the Traefik Web UI in the browser, you can use the “admin” NodePort 30729 (please note that your NodePort value might differ)  

#### Access traefik dashboard on browser http://localhost:30729

#### Step 5: Adding Ingress to the Cluster

##### Now we have Traefik as the Ingress Controller in the Kubernetes cluster. However, we still need to define the Ingress resource and a Service that exposes Traefik Web UI.

##### Let’s first create a Service:

kubectl create -f traefik-webui-svc.yaml

##### Let’s verify that the Service was created:


kubectl describe svc traefik-web-ui --namespace=kube-system

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

##### Next, we need to create an Ingress resource pointing to the Traefik Web UI backend.

kubectl create -f traefik-ingress.yaml  

##### You should now be able to see Traefik dashboard http://localhost:<admin_NodePort>
  
#### Step 6: Implementing Name-Based Routing  

##### Each Deployment will have two Pod replicas, and each Pod will serve the “animal” websites on the containerPort 80.

kubectl create -f animals-deployment.yaml

##### Now, let’s create a Service for each Deployment to make the Pods accessible:

kubectl create -f animals-svc.yaml

##### Finally, let’s create an Ingress with three frontend-backend pairs for each Deployment. bear.animal.com , moose.animal.com , and hare.animal.come will be our frontends pointing to corresponding backend Services.

kubectl create -f animals-ingress.yaml

##### Now, inside the Traefik dashboard and you should see a frontend for each host along with a list of corresponding backends.

##### If you edit your /etc/hosts again you should be able to access the animal websites in your browser.

vi /etc/hosts

10.32.0.3       bear.animal.com
10.32.0.4       bear.animal.com
10.40.0.6       hare.animal.com
10.32.0.5       hare.animal.com
10.40.0.5       moose.animal.com
10.40.0.4       moose.animal.com




curl bear.example.com

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

##### Access Traefik dashboard on browser at http://localhost:<admin_NodePort> 

##### To access created animal services 

http://bare.animal.com

http://hare.animal.com

http://moose.animal.com

#### Conclusion

##### Ingress is a powerful tool for routing external traffic to corresponding backend services in your Kubernetes cluster. Users can implement Ingress using a number of Ingress controllers supported by Kubernetes. Here we used Traefik Ingress controller that supports name-based routing, load balancing, and other common tasks of Ingress controllers. 


