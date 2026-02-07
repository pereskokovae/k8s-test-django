# Kubernetes & Infrastructure Setup

This guide describes two scenarios: `(1) Managed PostgreSQL with SSL (Yandex Cloud)` and `(2) local deployment in Minikube (Ingress + manifests)`.

---

## Managed PostgreSQL with SSL (Yandex Cloud)
This document describes how to prepare the development environment to connect to a Managed PostgreSQL instance in Yandex Cloud using SSL inside a Kubernetes cluster.

### Overview

- PostgreSQL is deployed outside the Kubernetes cluster (Managed PostgreSQL).
- Connection to the database requires an SSL CA certificate.
- The SSL certificate is stored in Kubernetes as a Secret.
- The certificate is mounted into a pod as a file, not baked into the Docker image.
- This approach allows changing the certificate without rebuilding images.


### 1. Create SSL Certificate Secret

First, download the PostgreSQL CA certificate from Yandex Cloud.

Then create a Kubernetes Secret containing the certificate:

```bash
kubectl create secret generic postgres-ssl \
  --from-file=root.crt=./root.crt \
  -n your_namespace #replace with your actual namespace
```

### 2. Test Pod with Mounted SSL Certificate

The pod manifest mounts the SSL certificate into the container
```yaml
volumeMounts:
  - name: pgssl
    mountPath: /certs
    readOnly: true

volumes:
  - name: pgssl
    secret:
      secretName: postgres-ssl
      items:
        - key: root.crt
          path: root.crt
```

This makes the certificate available inside the conteiner at:
```
/certs/root.crt
```

Apply the pod manifest
```bash
kubectl apply -f k8s/psql-pod.yaml -n your_namespace #replace with your actual namespace
```

### 3. Verify certificate inside the Pod

Connect to the Pod shell
```bash
kubectl exec -it psql-client -n <your-namespace> -- bash #replace with your actual namespace
```

Check than the certificate is mounted
```bash
ls -l /certs
```

Expected output
```
root.crt
```

### 4. Connect to PostgreSQL using SSL

Inside the Pod, connect to PostreSQL
```bash
psql "host=your_host port=your_port user=your_username dbname=your_db_name sslmode=require sslrootcert=/certs/root.crt" #replace with your actual data
```

### 5. Docker Image Build and Publish
Build the image (tagged with the current git commit SHA)
```bash
cd <project-root>
git rev-parse --short HEAD
docker login
docker build -t <your-dockerhub-username>/<repo>:<git-sha> ./backend_main_django
docker push <your-dockerhub-username>/<repo>:<git-sha>
```

### 6. Run Django in Kubernetes
The application is exposed via Nginx.

To access the site locally:
```bash
kubectl port-forward -n edu-ekaterina-pereskokova svc/main-nginx 8080:80
```

Open in browser: [http://localhost:8080](http://localhost:8080)

Django admin is available at: [http://localhost:8080/admin/](http://localhost:8080/admin/)

---

## Local deployment in Minikube

### 1. Start Minikube
1. Start the cluster
```bash
minikube start
kubectl config use-context minikube
```

### 2. Create application secret (Django)
How to create secret.yaml

The project expects sensitive configuration to be provided via a Kubernetes Secret.
1. Create a local file k8s/secret.yaml that will store sensitive data.

Example k8s/secret.yaml:
```bash
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgres://USER:PASSWORD@HOST:5432/DBNAME" # Replace with your values
  SECRET_KEY: "change_me" # Replace with your value
  ALLOWED_HOSTS: "127.0.0.1,localhost" # Replace with your values
```

2. Apply the secret
```bash
kubectl apply -f k8s/secret.yaml
```

### 3. PostgreSQL inside the cluster(Optional)
Install PostgreSQL inside the cluster (for example, using Helm) and define your own values

After PostgreSQL is installed, update the application secret with the correct DATABASE_URL
and restart the Django deployment
```bash
kubectl rollout restart deployment/django-deployment
```

### 4. Run database migrations
Apply database migrations and create a superuser account
```bash
kubectl exec -it deploy/django-deployment -- python manage.py migrate --noinput
kubectl exec -it deploy/django-deployment -- python manage.py createsuperuser
```

### 5. Ingress (accessing the site via a domain)
Ingress is used to expose the site via a domain name and the standard HTTP port 80,
without using an IP address in the URL.

This project uses NGINX Ingress in Minikube.

1. Enable Ingress in Minikube
```bash
minikube addons enable ingress
```
2. Apply the Ingress manifest
```bash
kubectl apply -f k8s/django-ingress.yaml
```

### 6. Configure /etc/hosts
The application will be available via the domain
```bash
star-burger.test
```
Get the Minikube IP address
```bash
minikube ip
```
Edit the /etc/hosts file and add the following line
```text
<MINIKUBE_IP> star-burger.test
```
Replace <MINIKUBE_IP> with the actual IP address returned by minikube ip.
After that, the site should be available at: [http://star-burger.test](http://star-burger.test)


## Clearing expired sessions (`clearsessions`)

This task is required to prevent the database from growing due to outdated user sessions and to keep the application stable over time.

The cleanup is implemented using a Kubernetes CronJob.

### Scheduled execution (CronJob)
The CronJob is configured to run periodically (for example, once per month).

1. Apply the CronJob manifest:
```bash
kubectl apply -f k8s/clearsession-cronjob.yaml
```

2. Verify that the CronJob has been created:
```bash
kubectl get cronjob
```

#### Run cleanup manually (one-time execution)
If needed, the cleanup task can be triggered manually by creating a Job from the existing CronJob.

1. Create a Job from the CronJob
```bash
kubectl create job --from=cronjob/django-clearsessions django-clearsessions-once
```

2. Verify that the Job has been created:
```bash
kubectl get jobs
```
