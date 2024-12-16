kubectl create namespace pikagrey
helm install ingress-nginx-pikagrey ingress-nginx/ingress-nginx -n pikagrey \
--set controller.ingressClassResource.name=nginx-pikagrey \
--set controller.ingressClassByName=true

helm upgrade ingress-nginx-pikagrey ingress-nginx/ingress-nginx -n pikagrey --set controller.ingressClassResource.name=nginx-pikagrey --set controller.ingressClassByName=true --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\\.io/os"=linux --set controller.admissionWebhooks.patch.failurePolicy=Ignore --set controller.scope.namespace=pikagrey


helm install pikagrey ./pikagrey -n pikagrey --create-namespace
kubectl port-forward svc/pikagrey 8085:5000 -n pikagrey
curl localhost:8085
helm upgrade pikagrey ./pikagrey -n pikagrey
helm uninstall pikagrey -n pikagrey