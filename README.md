To deploy a Laravel project on Kubernetes with a MySQL database, using **NodePort** for the Ingress and assigning a local domain name, follow these steps:

---

### Prerequisites
1. **Kubernetes Cluster**: Ensure a Kubernetes cluster is running and accessible.
2. **kubectl Installed**: Install and configure `kubectl` to manage the cluster.
3. **Docker Image**: Build and push your Laravel project as a Docker image to a container registry (like Docker Hub or a private registry).
4. **MySQL Instance**: Install MySQL as part of the Kubernetes deployment.
5. **Local DNS Configuration**: Map your local domain name to the cluster's NodePort.

---

### Deployment Steps

#### 1. **Create a Namespace**
```bash
kubectl create namespace laravel-app
kubectl config set-context --current --namespace=laravel-app
```

---

#### 2. **Create Persistent Volumes**
For Laravel storage and MySQL data:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: laravel-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Apply:
```bash
kubectl apply -f pvc.yaml
```

---

#### 3. **Deploy MySQL**
Create a MySQL Deployment and Service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: root_password
        - name: MYSQL_DATABASE
          value: laravel
        - name: MYSQL_USER
          value: laravel_user
        - name: MYSQL_PASSWORD
          value: laravel_password
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-data
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    app: mysql
  type: ClusterIP
```

Apply:
```bash
kubectl apply -f mysql-deployment.yaml
```

---

#### 4. **Deploy Laravel Application**
Create a Deployment and Service for Laravel:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel
spec:
  replicas: 1
  selector:
    matchLabels:
      app: laravel
  template:
    metadata:
      labels:
        app: laravel
    spec:
      containers:
      - name: laravel
        image: your-laravel-image:latest
        ports:
        - containerPort: 8000
        env:
        - name: DB_HOST
          value: mysql
        - name: DB_DATABASE
          value: laravel
        - name: DB_USERNAME
          value: laravel_user
        - name: DB_PASSWORD
          value: laravel_password
        volumeMounts:
        - name: laravel-storage
          mountPath: /var/www/html/storage
      volumes:
      - name: laravel-storage
        persistentVolumeClaim:
          claimName: laravel-storage
---
apiVersion: v1
kind: Service
metadata:
  name: laravel
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8000
    nodePort: 30080
  selector:
    app: laravel
```

Apply:
```bash
kubectl apply -f laravel-deployment.yaml
```

---

#### 5. **Configure Ingress**
Use Ingress to assign a local domain to your Laravel app:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: laravel-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: laravel.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: laravel
            port:
              number: 80
```

Apply:
```bash
kubectl apply -f ingress.yaml
```

---

#### 6. **Update Local DNS**
Update your local `/etc/hosts` file to resolve `laravel.local` to the Node IP:
```bash
<node-ip> laravel.local
```

---

#### 7. **Verify Deployment**
1. Check the status of pods and services:
   ```bash
   kubectl get pods
   kubectl get svc
   ```
2. Access the Laravel app via the browser using `http://laravel.local`.

---

### Troubleshooting
1. Ensure the **Ingress Controller** (e.g., Nginx) is installed in the cluster.
2. Verify DNS resolution using `curl`:
   ```bash
   curl -I http://laravel.local
   ```
3. Check logs if the app doesn’t work:
   ```bash
   kubectl logs <pod-name>
   ```

This setup deploys Laravel on Kubernetes with a MySQL database and NodePort ingress accessible via a local domain name.
