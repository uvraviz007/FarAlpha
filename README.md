# Flask + MongoDB on Kubernetes (Minikube Deployment)

This project deploys a Python Flask API connected to a MongoDB database on a Kubernetes cluster using Minikube.

---

## 1. Application Endpoints

- `/` → Returns welcome message + current time  
- `/data`
  - **POST** → Insert JSON  
  - **GET** → Fetch all documents  

---

## 2. Docker Build & Push

```bash
docker build -t uvraviz26/flask-mongo-app:v1 .
docker login
docker push uvraviz26/flask-mongo-app:v1
```

---

## 3. Kubernetes Files (Place inside `k8s/` folder)

### 3.1 Secret  
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

---

### 3.2 PV & PVC  
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

---

### 3.3 MongoDB StatefulSet  
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

---

### 3.4 MongoDB Service  
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
```

---

### 3.5 Flask Deployment  
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
          ports:
            - containerPort: 5000
```

---

### 3.6 Flask Service  
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

---

### 3.7 Horizontal Pod Autoscaler  
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

---

## 4. Deploy All Kubernetes Components

```bash
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo-pv-pvc.yaml
kubectl apply -f mongo-statefulset.yaml
kubectl apply -f mongo-service.yaml
kubectl apply -f flask-deployment.yaml
kubectl apply -f flask-service.yaml
kubectl apply -f flask-hpa.yaml
```

---

## 5. Accessing the Application

Use:

```bash
minikube service flask-service --url
```

This returns a URL like:

```
http://127.0.0.1:52296
```

Open it in a browser or Postman.

---

## 6. DNS Resolution (Short Explanation)

Kubernetes assigns each Service an internal DNS name:

```
<service-name>.<namespace>.svc.cluster.local
```

The Flask app connects to MongoDB using:

```
mongo-service
```

CoreDNS resolves this name to the correct MongoDB pod IP.

---

## 7. Resource Requests & Limits (Short Explanation)

- **Requests** = minimum guaranteed resources  
- **Limits** = maximum allowed resource usage  

Used configuration:

| Resource | Request | Limit |
|----------|---------|--------|
| CPU | 0.2 | 0.5 |
| Memory | 250Mi | 500Mi |

---

## 8. Autoscaling Test (Short)

Run load:

```bash
hey -z 60s -c 30 <flask_url>
```

Check autoscaler:

```bash
kubectl get hpa
kubectl get pods
```

Pods scale between **2 → 5** based on CPU usage.

---

# End of README.md
