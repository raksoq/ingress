helm install pikagrey ./pikagrey -n pikagrey
kubectl port-forward svc/pikagrey 8085:5000 -n pikagrey
curl localhost:8085
helm upgrade pikagrey ./pikagrey -n pikagrey