# Flask + MongoDB Application on Kubernetes (Minikube Deployment Guide)

This project demonstrates how to deploy a Python Flask API connected to a MongoDB database on a Kubernetes cluster.  
The application exposes two endpoints:

- `/` → Returns a welcome message with the current timestamp  
- `/data`
  - **POST** → Insert JSON data into MongoDB  
  - **GET** → Retrieve all MongoDB documents  

The setup includes Deployments, StatefulSets, Services, Persistent Volumes, Secrets, and Horizontal Pod Autoscaling.

---

# 1. Architecture Overview

The complete system consists of:

- **Flask Application Deployment**  
  Runs two replicas of the API server.

- **MongoDB StatefulSet**  
  Provides stable storage and authentication-enabled database.

- **Persistent Volume (PV) & Persistent Volume Claim (PVC)**  
  Ensures MongoDB data persists even if pods restart.

- **Kubernetes Secrets**  
  Stores MongoDB username and password securely.

- **Services**  
  - Flask: NodePort for external access  
  - MongoDB: ClusterIP for internal access only  

- **Horizontal Pod Autoscaler (HPA)**  
  Scales the Flask Deployment between 2–5 replicas based on CPU usage.

---

# 2. Prerequisites

Ensure the following tools are installed:

- Docker Desktop  
- Minikube  
- kubectl  
- Python 3.8+ (optional for local testing)

Your Docker Hub username used in this project: **uvraviz26**

---

# 3. Docker Build & Push Instructions

Inside the project folder:

```bash
docker build -t uvraviz26/flask-mongo-app:v1 .
docker login
docker push uvraviz26/flask-mongo-app:v1
```

---

# 4. Kubernetes Deployment Files

All YAML files should be stored in a `k8s/` directory.

---

## 4.1 MongoDB Credentials (Secret)

`mongo-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
type: Opaque
data:
  username: bW9uZ291c2Vy
  password: bW5nb3Bhc3M=
```

Apply:
```bash
kubectl apply -f mongo-secret.yaml
```

---

## 4.2 Persistent Volume & Persistent Volume Claim

`mongo-pv-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongo-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/mongo

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply:
```bash
kubectl apply -f mongo-pv-pvc.yaml
```

---

## 4.3 MongoDB StatefulSet

`mongo-statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: mongo-service
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: mongo:latest
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: username
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: password
          volumeMounts:
            - name: mongo-storage
              mountPath: /data/db
      volumes:
        - name: mongo-storage
          persistentVolumeClaim:
            claimName: mongo-pvc
```

Apply:
```bash
kubectl apply -f mongo-statefulset.yaml
```

---

## 4.4 MongoDB Service (Internal Only)

`mongo-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  type: ClusterIP
  selector:
    app: mongo
  ports:
    - port: 27017
      targetPort: 27017
```

Apply:
```bash
kubectl apply -f mongo-service.yaml
```

---

## 4.5 Flask Application Deployment

`flask-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask-app
          image: uvraviz26/flask-mongo-app:v1
          ports:
            - containerPort: 5000
          env:
            - name: MONGO_USER
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: username
            - name: MONGO_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: password
            - name: MONGO_HOST
              value: mongo-service
          resources:
            requests:
              cpu: "0.2"
              memory: "250Mi"
            limits:
              cpu: "0.5"
              memory: "500Mi"
```

Apply:
```bash
kubectl apply -f flask-deployment.yaml
```

---

## 4.6 Flask Service (External Access)

`flask-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  type: NodePort
  selector:
    app: flask-app
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 30080
```

Apply:
```bash
kubectl apply -f flask-service.yaml
```

---

## 4.7 Horizontal Pod Autoscaler

`flask-hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flask-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flask-app
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Apply:
```bash
kubectl apply -f flask-hpa.yaml
```

---

# 5. Accessing the Flask Application

Windows cannot access Minikube NodePort directly.  
Use Minikube's built-in service proxy:

```bash
minikube service flask-service --url
```

This provides a reachable URL such as:

```
http://127.0.0.1:52296
```

Open it in a browser or Postman.

---

# 6. DNS Resolution in Kubernetes

Kubernetes includes an internal DNS service (CoreDNS) that allows pods and services to communicate using names instead of IP addresses.

Every service gets a DNS name:

```
<service-name>.<namespace>.svc.cluster.local
```

In this setup:

- Flask connects to MongoDB using the hostname `mongo-service`
- CoreDNS resolves this name to the MongoDB ClusterIP  
- Even if pod IPs change, service discovery remains stable  

This ensures reliable inter-pod communication inside the cluster.

---

# 7. Resource Requests & Limits Explanation

### **Resource Requests**
Minimum guaranteed CPU/memory for a pod.  
The scheduler uses them to decide which node the pod fits on.

### **Resource Limits**
Defines the maximum CPU/memory a container is allowed to use.

### **Chosen configuration:**

| Resource | Request | Limit |
|---------|---------|--------|
| CPU     | 0.2     | 0.5    |
| Memory  | 250Mi   | 500Mi  |

These values ensure reliable performance while preventing resource exhaustion.

---

# 8. Horizontal Pod Autoscaler Testing

To test autoscaling, generate CPU load on the Flask endpoint:

Example using `hey`:

```bash
hey -z 60s -c 30 http://127.0.0.1:52296/
```

Monitor autoscaling:

```bash
kubectl get hpa
kubectl get pods
```

As CPU usage rises above 70%, HPA increases replicas up to a maximum of 5.

---

# 9. Design Choices

- **StatefulSet for MongoDB** → Stable identity and persistent storage  
- **ClusterIP for MongoDB** → Database is internal-only for security  
- **NodePort for Flask** → Allows external access during development  
- **Secrets for authentication** → Prevents hard-coding credentials  
- **Persistent Volume** → MongoDB data survives pod restarts  
- **Autoscaler** → Automatically adjusts capacity based on load  

---

# 10. Conclusion

This project demonstrates a complete microservice-style deployment on Kubernetes, including persistent storage, scaling, networking, and secure configuration management.  
The setup follows production-ready principles while remaining simple enough for learning and testing.

---

# ✔ End of README.md
