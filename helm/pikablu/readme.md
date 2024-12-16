helm uninstall ingress-nginx-pikablu -n pikablu

helm install ingress-nginx-pikablu ingress-nginx/ingress-nginx -n pikablu \
--set controller.ingressClassResource.name=nginx-pikablu \
--set controller.ingressClassByName=true

helm upgrade ingress-nginx-pikablu ingress-nginx/ingress-nginx -n pikablu --set controller.ingressClassResource.name=nginx-pikablu --set controller.ingressClassByName=true --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\\.io/os"=linux --set controller.admissionWebhooks.patch.failurePolicy=Ignore --set controller.scope.namespace=pikablu



helm install pikablu ./pikablu -n pikablu --create-namespace
kubectl port-forward svc/pikablu 8085:5000 -n pikablu
curl localhost:8085
helm upgrade pikablu ./pikablu -n pikablu
helm uninstall pikablu -n pikablu
