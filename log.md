 # Logs from installing ingress

 It works both by helm and kubectl
 ingress in different namespace
 ingress uses external IP even not specified in kubectl get ingress

 ```bash

lapek@Oskars-MacBook-Air helm % k get all -n ingress-nginx
NAME                                                       READY   STATUS    RESTARTS   AGE
pod/my-ingress-ingress-nginx-controller-6b57fffd67-bslsl   1/1     Running   0          61s

NAME                                                    TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                      AGE
service/my-ingress-ingress-nginx-controller             LoadBalancer   10.0.173.10   74.248.109.143   80:30192/TCP,443:31309/TCP   62s
service/my-ingress-ingress-nginx-controller-admission   ClusterIP      10.0.29.101   <none>           443/TCP                      62s

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-ingress-ingress-nginx-controller   1/1     1            1           62s

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/my-ingress-ingress-nginx-controller-6b57fffd67   1         1         1       62s
lapek@Oskars-MacBook-Air helm % helm install pikablu ./pikablu -n pikablu --create-namespace
NAME: pikablu
LAST DEPLOYED: Wed Dec 18 10:49:41 2024
NAMESPACE: pikablu
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  http:///
  
lapek@Oskars-MacBook-Air helm % k get ingress -A                                          
NAMESPACE   NAME      CLASS   HOSTS   ADDRESS   PORTS   AGE
pikablu     pikablu   nginx   *                 80      5s

lapek@Oskars-MacBook-Air helm % curl 74.248.109.143/health 
{"status":"ok"}
 ```

## change ingress=false

 ```bash
lapek@Oskars-MacBook-Air helm % helm upgrade pikablu ./pikablu -n pikablu --create-namespace
Release "pikablu" has been upgraded. Happy Helming!
NAME: pikablu
LAST DEPLOYED: Wed Dec 18 10:50:56 2024
NAMESPACE: pikablu
STATUS: deployed
REVISION: 2
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace pikablu -l "app.kubernetes.io/name=pikablu,app.kubernetes.io/instance=pikablu" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace pikablu $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace pikablu port-forward $POD_NAME 8080:$CONTAINER_PORT
lapek@Oskars-MacBook-Air helm % k get ingress -A                                            
No resources found
lapek@Oskars-MacBook-Air helm % curl 74.248.109.143/health                                  
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
 ```
# add kubectl ingress

 ```bash
lapek@Oskars-MacBook-Air helm % kubectl apply -f ../pikablu-ingress.yaml -n pikablu   
ingress.networking.k8s.io/pikablu-ingress created
lapek@Oskars-MacBook-Air helm % k get ingress -A                                      
NAMESPACE   NAME              CLASS   HOSTS   ADDRESS   PORTS   AGE
pikablu     pikablu-ingress   nginx   *                 80      3s
lapek@Oskars-MacBook-Air helm % curl 74.248.109.143/health                            
{"status":"ok"}
lapek@Oskars-MacBook-Air helm % 
 ```