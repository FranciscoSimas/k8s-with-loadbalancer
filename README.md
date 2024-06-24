## Kubernetes Deployment Project Documentation

## Project Overview

This project aims to deploy several applications on a Kubernetes cluster using YAML configurations. The applications included are EJBCA, One Time Secret, Plik, Wiki.js, and PostgreSQL. Additionally, an Nginx proxy is used to handle SSL termination and route traffic to the appropriate services. The SSL certificates are obtained from Let's Encrypt and the DNS domains are configured using CloudNS.

## Prerequisites

Before deploying the project, ensure you have the following prerequisites:

1. Kubernetes cluster (minikube, k3s, or a managed Kubernetes service like GKE, EKS, or AKS).
2. `kubectl` CLI tool installed and configured to interact with your Kubernetes cluster.
3. `certbot` installed for generating SSL certificates from Let's Encrypt.
4. DNS records configured for the domains used in the Nginx configuration.

## Project Structure

The project directory contains the following files:

- `deploy-all.sh`: A shell script to apply all Kubernetes configurations in sequence.
- `postgres-deployment.yaml`: Deployment and service configuration for PostgreSQL.
- `plik-deployment.yaml`: Deployment and service configuration for Plik.
- `wiki-deployment.yaml`: Deployment and service configuration for Wiki.js.
- `onetimesecret-deployment.yaml`: Deployment and service configuration for One Time Secret.
- `ejbca-deployment.yaml`: Deployment and service configuration for EJBCA.
- `nginx-configmap.yaml`: ConfigMap for Nginx configuration.
- `nginx-deployment.yaml`: Deployment and service configuration for Nginx.

## Step-by-Step Deployment Guide

### 1. Generate SSL Certificates

Use Certbot to generate SSL certificates for your domains. Replace `yourdomain.com` with your actual domain names.

```bash
sudo certbot certonly --manual --preferred-challenges=dns -d secret.simas.cloudns.ch -d plik.simas.cloudns.ch -d wiki.simas.cloudns.ch -d cert.simas.cloudns.ch
```

### 2. Prepare Kubernetes Secrets for SSL Certificates

After generating the certificates, create a Kubernetes secret to store them. Replace the paths with the actual paths to your certificate files.

```bash
kubectl create secret generic simas14secret --from-file=fullchain.pem=/etc/letsencrypt/live/yourdomain.com/fullchain.pem --from-file=privkey.pem=/etc/letsencrypt/live/yourdomain.com/privkey.pem
```

### 3. Deploy Services

Run the deployment script to apply all Kubernetes configurations.

```bash
chmod +x deploy-all.sh
./deploy-all.sh
```

### 4. Verification

After deploying the services, verify that all pods are running correctly.

```bash
kubectl get pods
```

Also, verify that the services are running and accessible via their respective URLs.

## Detailed YAML Configurations

### EJBCA Deployment (`ejbca-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ejbca-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ejbca
  template:
    metadata:
      labels:
        app: ejbca
    spec:
      containers:
      - name: ejbca
        image: primekey/ejbca-ce:latest
        ports:
        - containerPort: 8080
        - containerPort: 8443

---
apiVersion: v1
kind: Service
metadata:
  name: ejbca-service
spec:
  selector:
    app: ejbca
  ports:
    - name: http
      protocol: TCP
      port: 7080
      targetPort: 8080
    - name: https
      protocol: TCP
      port: 8443
      targetPort: 8443
  type: ClusterIP
```

### Nginx ConfigMap (`nginx-configmap.yaml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
        listen 80;
        server_name secret.simas.cloudns.ch;

        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl;
        server_name secret.simas.cloudns.ch;
        ssl_certificate /etc/nginx/certs/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/privkey.pem;
        
        location / {
            proxy_pass http://secret-service.default.svc.cluster.local:7143;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 80;
        server_name plik.simas.cloudns.ch;

        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl;
        server_name plik.simas.cloudns.ch;
        
        ssl_certificate /etc/nginx/certs/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/privkey.pem;
        
        location / {
            proxy_pass http://plik-service.default.svc.cluster.local:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 80;
        server_name wiki.simas.cloudns.ch;

        location / {
            return 301 https://$host$request_uri;
        }
    }
    
    server {
        listen 443 ssl;
        server_name wiki.simas.cloudns.ch;
        ssl_certificate /etc/nginx/certs/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/privkey.pem;

        location / {
            proxy_pass http://wiki-service.default.svc.cluster.local:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 80;
        server_name cert.simas.cloudns.ch;

        location / {
            rewrite ^/(.*)$ https://$server_name/ejbca/adminweb/$1 permanent;
        }
    }

    server {
        listen 443 ssl;
        server_name cert.simas.cloudns.ch;

        ssl_certificate /etc/nginx/certs/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/privkey.pem;

        location /ejbca/adminweb {
            proxy_pass https://ejbca-service.default.svc.cluster.local:8443;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
```

### Nginx Deployment (`nginx-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-proxy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        - containerPort: 443
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        - name: nginx-certs
          mountPath: /etc/nginx/certs
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: nginx-certs
        secret:
          secretName: simas14secret

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
    - name: https
      protocol: TCP
      port: 443
      targetPort: 443
      nodePort: 30443
  type: LoadBalancer
```

### One Time Secret Deployment (`onetimesecret-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secret
  template:
    metadata:
      labels:
        app: secret
    spec:
      containers:
      - name: secret
        image: dismantl/onetimesecret:latest
        ports:
        - containerPort: 7143
---
apiVersion: v1
kind: Service
metadata:
  name: secret-service
spec:
  selector:
    app: secret
  ports:
    - protocol: TCP
      port: 7143
      targetPort: 7143
  type: ClusterIP
```

### Plik Deployment (`plik-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: plik-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: plik
  template:
    metadata:
      labels:
        app

: plik
    spec:
      containers:
      - name: plik
        image: rootgg/plik:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: plik-service
spec:
  selector:
    app: plik
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

### PostgreSQL Deployment (`postgres-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:latest
        env:
          - name: POSTGRES_DB
            value: plik_db
          - name: POSTGRES_USER
            value: plik_user
          - name: POSTGRES_PASSWORD
            value: password  # Replace with a secure password
        ports:
          - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
```

### Wiki.js Deployment (`wiki-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wiki-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wiki
  template:
    metadata:
      labels:
        app: wiki
    spec:
      containers:
      - name: wiki
        image: linuxserver/wikijs:latest
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: wiki-service
spec:
  selector:
    app: wiki
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
  type: ClusterIP
```

### Deployment Script (`deploy-all.sh`)

```bash
#!/bin/bash

sleep 2
kubectl apply -f postgres-deployment.yaml

sleep 2
kubectl apply -f plik-deployment.yaml

sleep 2
kubectl apply -f wiki-deployment.yaml

sleep 2
kubectl apply -f onetimesecret-deployment.yaml

sleep 2
kubectl apply -f ejbca-deployment.yaml

sleep 2
kubectl apply -f nginx-configmap.yaml

sleep 2
kubectl apply -f nginx-deployment.yaml
```

## Conclusion

This documentation provides a comprehensive guide for deploying multiple applications on a Kubernetes cluster using YAML configurations. Follow the steps carefully to ensure a successful deployment. For any issues or improvements, feel free to raise an issue or contribute to the repository.
