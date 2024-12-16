# ingress

## Conclusions:

1. yaml ingress works as well as helm ingress if specified correctly.
2. pikablu and ingress in the same namespace works 
3. ingress in ingress-nginx and pikablu/pikagrey work when pikablu is implemented as / and pikagrey and /pikagrey

 ```yaml
ingress:
  enabled: true
  className: nginx
  annotations: 
    # nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: "" 
      paths:
        - path: /
          pathType: Prefix
          # backend:
          #   serviceName: pikablu  
          #   servicePort: 80 

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  hosts:
    - host: ""  
      paths:
        - path: /pikagrey(/|$)(.*)
          pathType: ImplementationSpecific
          backend:
            service:
              name: pikagrey
              port:
                number: 80
 ```

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
 ```

 ```bash
helm install my-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer
 ```

### it works only in pikablu namespace fo pikablu service
 ```bash
kubectl apply -f pikablu-ingress.yaml -n pikablu   
 ```

#### values.yaml

```yaml
 service:
  type: ClusterIP
  port: 80
  targetPort: 5000

# works curl ingress-lba-IP without /pikablu but with clusterIP service
ingress:
  enabled: false
  ingressClassName: nginx
  annotations:
    # nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: ""  
      paths:
        - path: /
          pathType: Prefix
```
