# Deployment Plan for a Django Application on Kubernetes
### prerequisites
# Kubernetes Cluster Required Ports

When setting up a **Kubernetes cluster on AWS EC2**, you must ensure the necessary ports are open to allow proper communication between control plane, worker nodes, and external clients. If operation on other service providers, be sure to confirm if the firewall does not block the said ports. 

## 📌 Required Ports

| **Component**            | **Protocol** | **Ports**        | **Purpose** |
|--------------------------|-------------|-----------------|-------------|
| **Kubernetes API Server** | TCP         | 6443            | Allows communication with the Kubernetes API server. Required for `kubectl` commands. |
| **etcd**                 | TCP         | 2379-2380       | Stores cluster data. Only required on control plane nodes. |
| **Kubelet API**          | TCP         | 10250           | Enables `kubectl logs`, `exec`, and health checks on worker nodes. |
| **NodePort Services**    | TCP/UDP     | 30000-32767     | Exposes services externally when using NodePort type. |
| **kube-scheduler**       | TCP         | 10251           | Handles scheduling of pods on nodes. |
| **controller-manager**   | TCP         | 10252           | Manages controllers in the cluster. |
| **Ingress Controller**   | TCP         | 80, 443         | Allows external traffic via HTTP/HTTPS when using an ingress controller. |
| **SSH (Optional)**       | TCP         | 22              | Enables SSH access to EC2 instances. Should be restricted to your IP only. |
| **CoreDNS (Internal DNS)** | UDP       | 53              | Handles DNS resolution within the cluster. |
| **VXLAN (Flannel CNI)**  | TCP/UDP     | 4789            | Required for networking between pods when using Flannel CNI. |
| **IPsec (Calico CNI)**   | TCP/UDP     | 4500, 500       | Used for secure pod-to-pod networking in Calico. |

## 🔧 How to Update Security Groups on AWS
1. Go to **AWS Console** → **EC2** → **Security Groups**.
2. Find the **security group** associated with your Kubernetes cluster.
3. Click **Inbound Rules** → **Edit inbound rules**.
4. Add the required **ports & protocols** based on the table above.
5. Click **Save rules**.



1. Build and push your Django image.
- Ensure the Django Database configurations are well set before creating a build
```python
import os

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': os.getenv('DATABASE_NAME', 'mydb'),
        'USER': os.getenv('DATABASE_USER', 'database'),
        'PASSWORD': os.getenv('DATABASE_PASSWORD', 'secret'),
        'HOST': os.getenv('DATABASE_HOST', 'mysql'),  # Service name in Kubernetes
        'PORT': os.getenv('DATABASE_PORT', '3306'),
        'OPTIONS': {
            'init_command': "SET sql_mode='STRICT_TRANS_TABLES'"
        }
    }
}

```
- Understand the parameters used in the above
```
DATABASE_NAME → The database name (mydb in our case).
DATABASE_USER → The MySQL user from Secrets.
DATABASE_PASSWORD → The password from Secrets.
DATABASE_HOST → The MySQL Service name (mysql).
DATABASE_PORT → Default MySQL port (3306).
OPTIONS → Ensures strict mode is enabled for MySQL.
```
```dockerfile
# Dockerfile
FROM python:3.10
WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "myproject.wsgi:application"]
```
- Build the image
```sh
docker build -t my-django-app .
```
- Push it to a registry
```sh
docker tag my-django-app my-registry/my-django-app:v1
docker push my-registry/my-django-app:v1
```
2. Persistent Volume (PV) & Persistent Volume Claim (PVC) for MySQL
- MySQL needs persistent storage, to ensure data is not lost should a mysql pod be recreated.
```yaml
# mysql-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/mysql"

---
# mysql-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

3. Secret for MySQL Credentials
- create a secret for mysql database.
```yaml
# mysql-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: cGFzc3dvcmQ=   # base64-encoded "password"
  MYSQL_USER: ZGF0YWJhc2U=            # base64-encoded "database"
  MYSQL_PASSWORD: c2VjcmV0            # base64-encoded "secret"
```

4. MySQL Deployment & Service
```yaml
# mysql-deployment.yaml
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
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              value: mydb
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          ports:
            - containerPort: 3306
          volumeMounts:
            - mountPath: /var/lib/mysql
              name: mysql-storage
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc

---
# mysql-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306

```

5. Django ConfigMap
```yaml
# django-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
data:
  DATABASE_HOST: "mysql"
  DATABASE_NAME: "mydb"
  DATABASE_USER: "database"
```

6. Django Deployment & Service
```yaml
# django-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      containers:
        - name: django
          image: my-registry/my-django-app:v1
          env:
            - name: DATABASE_HOST
              valueFrom:
                configMapKeyRef:
                  name: django-config
                  key: DATABASE_HOST
            - name: DATABASE_NAME
              valueFrom:
                configMapKeyRef:
                  name: django-config
                  key: DATABASE_NAME
            - name: DATABASE_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          ports:
            - containerPort: 8000

---
# django-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: django
spec:
  selector:
    app: django
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: NodePort

```

7. Ingress & Domain Configuration
Ensure you have an Ingress Controller (e.g., NGINX) installed:
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```
Then create the Ingress Resource:
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: django-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: mydjangoapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: django
                port:
                  number: 8000
```

8. Apply All Configurations
```sh
kubectl apply -f mysql-pv.yaml
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f django-config.yaml
kubectl apply -f django-deployment.yaml
kubectl apply -f django-service.yaml
kubectl apply -f ingress.yaml
```

9. Verify Database Connection
After deploying your Django app, you can test the connection inside your pod:
```sh
kubectl exec -it <django-pod-name> -- python manage.py dbshell
```
- Do migration
```sh
kubectl exec -it <django-pod-name> -- python manage.py migrate
```


10. Scaling & Enhancements
- Add Horizontal Pod Autoscaler (HPA):
```sh
kubectl autoscale deployment django --cpu-percent=50 --min=2 --max=5
```

11. Enforce SSL certificate
- Will update


- Update settings.py for your django app for force HTTPS
```python
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
```
