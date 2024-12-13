helm install pikaflask ./pikaflask -n pika
kubectl port-forward svc/pikaflask 8080:80 -n pika