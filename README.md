# traefik

kubectl create -f traefik-rbac.yaml

kubectl create -f trefik-daemonset.yaml

### Create NodePorts for External Access

kubectl create -f traefik-svc.yaml

kubectl get pods --all -n kube-system | grep traefik

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


kubectl create -f animals-deployment.yaml

kubectl create -f animals-svc.yaml

kubectl create -f animals-ingress.yaml

## Access the UI at http://localhost:<admin_port> 
