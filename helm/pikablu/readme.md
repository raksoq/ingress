helm install pikablu ./pikablu -n pikablu 
kubectl port-forward svc/pikablu 8085:5000 -n pikablu
curl localhost:8085
helm upgrade pikablu ./pikablu -n pikablu
helm uninstall pikablu -n pikablu