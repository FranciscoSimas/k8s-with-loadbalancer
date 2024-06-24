
---

## Kubernetes Deployment Project Documentation

## Project Overview

This project aims to deploy several applications on a Kubernetes cluster using YAML configurations. The applications included are EJBCA, One Time Secret, Plik, Wiki.js, and PostgreSQL. Additionally, an Nginx proxy is used to handle SSL termination and route traffic to the appropriate services. The SSL certificates are obtained from Let's Encrypt and the DNS domains are configured using CloudNS.

## Prerequisites

Before deploying the project, ensure you have the following prerequisites:

1. Kubernetes cluster (minikube, k3s, or a managed Kubernetes service like GKE, EKS, or AKS).
2. `kubectl` CLI tool installed and configured to interact with your Kubernetes cluster.
3. DNS records configured for the domains used in the Nginx configuration.

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

### 1. Install Necessary Tools

Install the required development tools and Certbot:

```bash
sudo yum groupinstall "Development tools"
sudo yum install certbot
```

### 2. Configure DNS on CloudNS

1. Go to [CloudNS](https://www.cloudns.net/main/).
2. Create a new DNS zone with your desired domain name.
3. Within the zone, create 4 host records for each service (One Time Secret, Plik, Wiki.js, EJBCA) with type A, pointing to the public IP of your Kubernetes control plane.

### 3. Generate SSL Certificates

Use Certbot to generate SSL certificates for your domains:

```bash
sudo certbot certonly --manual --preferred-challenges=dns -d secret.yourdomain.com -d plik.yourdomain.com -d wiki.yourdomain.com -d cert.yourdomain.com
```

### 4. Prepare Kubernetes Secrets for SSL Certificates

Create a directory named `certs` in your project folder and copy the SSL certificates to this directory:

```bash
sudo su
cd /etc/letsencrypt/live
cd [yourdomain.com]
cp fullchain.pem /home/ec2-user/[your-folder]/certs
cp privkey.pem /home/ec2-user/[your-folder]/certs
chown ec2-user:ec2-user /home/ec2-user/[your-folder]/certs/fullchain.pem /home/ec2-user/[your-folder]/certs/privkey.pem
exit
```

Create a Kubernetes secret to store the SSL certificates:

```bash
kubectl create secret generic "yourname"secret --from-file=fullchain.pem=/home/ec2-user/[your-folder]/certs/fullchain.pem --from-file=privkey.pem=/home/ec2-user/[your-folder]/certs/privkey.pem
```

### 5. Update DNS Records

1. Go to [CloudNS](https://www.cloudns.net/main/).
2. In the same DNS zone, delete the existing A records.
3. Create new CNAME records for the same hostnames, pointing to the LoadBalancer.

### 6. Deploy Services

Run the deployment script to apply all Kubernetes configurations:

```bash
sh deploy-all.sh
```

### 7. Verify Deployments

Check that all pods are running correctly:

```bash
kubectl get all
```

## Conclusion

This documentation provides a comprehensive guide for deploying multiple applications on a Kubernetes cluster using YAML configurations. Follow the steps carefully to ensure a successful deployment. For any issues or improvements, feel free to raise an issue or contribute to the repository.

---
