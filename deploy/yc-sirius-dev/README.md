# Dev Environment Setup (PostgreSQL with SSL)

This document describes how to prepare the development environment to connect to a Managed PostgreSQL instance in Yandex Cloud using SSL inside a Kubernetes cluster.

---

## Overview

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

