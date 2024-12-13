# Solution Using Ingress and ClusterIP Service

This document describes the solution for managing your **pikagrey** and **pikablu** applications in Kubernetes using **Ingress** with **ClusterIP** services. It also explains the problems you encountered with target ports and ingress rewrite rules and details how pods, services, and ingress resources are connected.

---

## **Application Code for pikagrey and pikablu**

Both **pikagrey** and **pikablu** applications are Flask-based web applications, exposing a web server on port `5000` with the following routes:

### Sample Application Code
```python
from flask import Flask, jsonify, render_template

app = Flask(__name__)

@app.route('/')
def home():
    return render_template("index.html")

@app.route('/health')
def health():
    return jsonify({"status": "ok"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Build the Docker images:
```bash
docker build -t oskarq/pikaflask:v3 .  # for pikagrey
docker build -t oskarq/pikaflask:v1 .  # for pikablu
```

Push them to your registry:
```bash
docker push oskarq/pikaflask:v3
docker push oskarq/pikaflask:v1
```

---

## **Helm Chart Values.yaml for pikagrey and pikablu**

Each application has a **ClusterIP** service to route traffic to the pods and an ingress resource to manage external access.

### Values.yaml for pikagrey
```yaml
replicaCount: 1

image:
  repository: oskarq/pikaflask
  pullPolicy: IfNotPresent
  tag: v3

service:
  type: ClusterIP
  port: 80
  targetPort: 5000

ingress:
  enabled: true
  className: nginx
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: ""
      paths:
        - path: /pikagrey
          pathType: Prefix
```

### Values.yaml for pikablu
```yaml
replicaCount: 1

image:
  repository: oskarq/pikaflask
  pullPolicy: IfNotPresent
  tag: v1

service:
  type: ClusterIP
  port: 80
  targetPort: 5000

ingress:
  enabled: true
  className: nginx
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: ""
      paths:
        - path: /pikablu
          pathType: Prefix
```

---

## **Problem: Target Port and Rewrite Rules**

### **Target Port Issue**
Your service's `targetPort` was initially set to `http` in the Helm template:
```yaml
ports:
  - port: {{ .Values.service.port }}
    targetPort: http
    protocol: TCP
    name: http
```
However, your Flask application listens on `5000`, and the container does not define a named port `http`. This mismatch caused the **connection refused** errors.

#### **Solution**
Update the Helm template to use the numeric `targetPort` from `values.yaml`:
```yaml
ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
    protocol: TCP
    name: http
```
This ensures traffic from the service is correctly forwarded to port `5000` inside the pod.

---

### **Rewrite Rules in Ingress**
Your ingress initially routed `/pikagrey` and `/pikablu` directly to the backend services without rewriting the path. Since the Flask app only serves `/` and `/health`, requests to `/pikagrey` and `/pikablu/health` failed with **404 Not Found** errors.

#### **Solution**
Add the following annotation to the ingress to strip the `/pikagrey` or `/pikablu` prefix before forwarding:
```yaml
ingress:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```
This rewrites requests from `/pikagrey/health` to `/health` for the backend service.

---

## **How Pods, Services, and Ingress are Connected**

### **Pod**
- Pods run the containerized application and expose the Flask server on `5000`.
- Pod labels are used to identify the pod for the service.

### **Service**
- A `ClusterIP` service exposes the pod to other resources in the cluster.
- The service forwards traffic from `port: 80` to `targetPort: 5000`.
- The service selector matches the pod labels to route traffic correctly.

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: pikagrey
  labels:
    app.kubernetes.io/name: pikagrey
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 5000
  selector:
    app.kubernetes.io/name: pikagrey
```

### **Ingress**
- The ingress handles external HTTP traffic and forwards it to the service based on the path.
- Paths like `/pikagrey` are rewritten to `/` for the backend service.

**Example:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pikagrey-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: ""
      http:
        paths:
          - path: /pikagrey
            pathType: Prefix
            backend:
              service:
                name: pikagrey
                port:
                  number: 80
```

---

## **Testing the Setup**

1. Deploy the Helm charts for `pikagrey` and `pikablu`:
   ```bash
   helm upgrade --install pikagrey ./pikagrey -n pikagrey
   helm upgrade --install pikablu ./pikablu -n pikablu
   ```

2. Verify the services:
   ```bash
   kubectl get svc -n pikagrey
   kubectl get svc -n pikablu
   ```

3. Test ingress routing:
   ```bash
   curl http://localhost/pikagrey
   curl http://localhost/pikagrey/health
   curl http://localhost/pikablu
   curl http://localhost/pikablu/health
   ```

**Expected Results:**
- `/pikagrey` should return the pikagrey app homepage.
- `/pikagrey/health` should return `{"status": "ok"}`.
- `/pikablu` should return the pikablu app homepage.
- `/pikablu/health` should return `{"status": "ok"}`.

---

## **Conclusion**
By configuring **ClusterIP** services and using **Ingress** with rewrite rules, you successfully manage multiple versions of your applications (`pikagrey` and `pikablu`) with clean routing. This setup avoids conflicts and ensures the backend services receive traffic in the expected format.


# Explanation: Why `nginx.ingress.kubernetes.io/rewrite-target: /$2` Works

## **Updated Ingress Configuration**
Your working configuration:
```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    kubernetes.io/ingress.class: nginx
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

## **Key Components**
### **1. Path Matching**
- `path: /pikagrey(/|$)(.*)`:
  - `/pikagrey(/|$)` matches `/pikagrey` followed by either:
    - `/` (a trailing slash), or
    - `$` (the end of the string).
  - `(.*)` captures the remainder of the path after `/pikagrey`.

#### Examples of Matching:
- `/pikagrey` → `$2` is empty.
- `/pikagrey/` → `$2` is empty.
- `/pikagrey/health` → `$2` is `health`.

---

### **2. Rewrite Target**
- `nginx.ingress.kubernetes.io/rewrite-target: /$2`:
  - This annotation rewrites the incoming request path using the captured group `$2`.
  - If `$2` is empty, the rewrite target becomes `/`.
  - If `$2` contains a value (e.g., `health`), the rewrite target becomes `/$2` (e.g., `/health`).

---

### **3. ImplementationSpecific PathType**
- `pathType: ImplementationSpecific`:
  - Enables advanced path matching with regular expressions in NGINX ingress.
  - Ensures compatibility with the rewrite rules for flexible routing.

---

## **How It Works**

### Request to `/pikagrey/health`
1. The request matches the regex `/pikagrey(/|$)(.*)`.
2. `$2` is captured as `health`.
3. The rewrite rule transforms `/pikagrey/health` into `/health`.
4. The backend service receives the rewritten path `/health`.

---

### Request to `/pikagrey/` or `/pikagrey`
1. The request matches `/pikagrey(/|$)(.*)`.
2. `$2` is empty.
3. The rewrite rule transforms `/pikagrey` into `/`.
4. The backend service receives the rewritten path `/`.

---

### Request to `/pikagrey/somethingelse`
1. The request matches `/pikagrey(/|$)(.*)`.
2. `$2` is captured as `somethingelse`.
3. The rewrite rule transforms `/pikagrey/somethingelse` into `/somethingelse`.
4. The backend service receives the rewritten path `/somethingelse`.

---

## **Why This Fixes the Problem**

In the previous configuration:
```yaml
inginx.ingress.kubernetes.io/rewrite-target: /
```
- All requests, regardless of the path, were rewritten to `/`.
- This caused the backend to always receive `/`, resulting in the root page being served even for non-existent paths like `/pikagrey/health`.

In the updated configuration:
```yaml
inginx.ingress.kubernetes.io/rewrite-target: /$2
```
- Only the path after `/pikagrey` is forwarded to the backend.
- For example:
  - `/pikagrey/health` becomes `/health`.
  - `/pikagrey/somethingelse` becomes `/somethingelse`.

This ensures:
1. Proper routing to valid paths (`/health`).
2. The backend receives the appropriate path for processing.
3. Undefined paths (e.g., `/pikagrey/invalid`) can return `404` from the backend.

---

## **Benefits of the Updated Configuration**
1. **Dynamic Path Handling**:
   - The `$2` placeholder ensures that only the relevant portion of the path is forwarded to the backend.

2. **Flexibility**:
   - Allows matching of both the root path (`/pikagrey`) and subpaths (`/pikagrey/health`, `/pikagrey/other`).

3. **Correct Behavior**:
   - Prevents the root page (`/`) from being served for undefined routes.

---



