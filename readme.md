# Deploying YugabyteDB and a Node.js Application with Monitoring

## Overview

This project involves deploying YugabyteDB and a Node.js web application on Kubernetes. YugabyteDB will be managed using StatefulSet for data persistence and high availability. The Node.js application will be deployed using Deployment. Additionally, we will set up monitoring for the database and application using Prometheus.

## Objectives

1.  **Deploy YugabyteDB** using StatefulSet for high availability and persistence.
2.  **Deploy a Node.js application** using Deployment for scalability and load balancing.
3.  **Monitor the system** using Prometheus and Grafana.
4.  **Expose services** using Kubernetes Services and Ingress for external access.

## 1. Installation

**Prerequisites:**

-   Kubernetes cluster (e.g., Minikube, cloud provider, etc.)
-   `kubectl` CLI tool installed
-   Docker installed

### 1.1 Install Minikube (Optional, if you don’t have a Kubernetes cluster)

Follow the official Minikube installation guide to install Minikube on your system.

### 1.2 Start Minikube


```bash
minikube start
``` 

## 2. Kubernetes Configuration

### 2.1 YugabyteDB StatefulSet and Service

**File: `yugabyte-db-statefulset.yaml`**


```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: yugabyte-db
spec:
  serviceName: "yugabyte-db"
  replicas: 3
  selector:
    matchLabels:
      app: yugabyte-db
  template:
    metadata:
      labels:
        app: yugabyte-db
    spec:
      containers:
      - name: yugabyte
        image: yugabytedb/yugabyte:latest
        ports:
        - containerPort: 5433
        volumeMounts:
        - name: data
          mountPath: /mnt/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
``` 

**File: `yugabyte-db-service.yaml`**


```yaml
apiVersion: v1
kind: Service
metadata:
  name: yugabyte-db
spec:
  ports:
  - port: 5433
    targetPort: 5433
  clusterIP: None
  selector:
    app: yugabyte-db
``` 

### 2.2 Node.js Application Deployment and Service

**File: `nodejs-app-deployment.yaml`**


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
      - name: nodejs-app
        image: your-nodejs-app-image:latest
        ports:
        - containerPort: 3000
        env:
        - name: DB_HOST
          value: yugabyte-db
        - name: DB_PORT
          value: "5433"
        - name: DB_NAME
          value: mydatabase
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: password
``` 

**File: `nodejs-app-service.yaml`**


```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodejs-app-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
  selector:
    app: nodejs-app
``` 

### 2.3 Prometheus Configuration

**File: `prometheus-config.yaml`**


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'nodejs-app'
        static_configs:
          - targets: ['nodejs-app-service:3000']
      - job_name: 'yugabyte-db'
        static_configs:
          - targets: ['yugabyte-db:5433']
``` 

**File: `prometheus-deployment.yaml`**


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
          subPath: prometheus.yml
      volumes:
      - name: config
        configMap:
          name: prometheus-config
``` 

**File: `prometheus-service.yaml`**


```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
spec:
  ports:
  - port: 9090
  selector:
    app: prometheus
``` 

### 2.4 Grafana Configuration

**File: `grafana-deployment.yaml`**


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
``` 

**File: `grafana-service.yaml`**


```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
spec:
  type: LoadBalancer
  ports:
  - port: 3000
  selector:
    app: grafana
``` 

### 2.5 Ingress Configuration

**File: `nodejs-app-ingress.yaml`**


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nodejs-app-ingress
spec:
  rules:
  - host: nodejs-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nodejs-app-service
            port:
              number: 80
``` 

### 2.6 Secrets

**File: `db-secrets.yaml`**


```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
type: Opaque
data:
  username: <base64-encoded-username>
  password: <base64-encoded-password>
``` 

_Note: Replace `<base64-encoded-username>` and `<base64-encoded-password>` with base64 encoded values._

## 3. Apply the Configuration

1.  **Deploy YugabyteDB StatefulSet:**

    
    ```bash
    kubectl apply -f yugabyte-db-statefulset.yaml
    ``` 
    
2.  **Create YugabyteDB Service:**

    
    ```bash
    kubectl apply -f yugabyte-db-service.yaml
    ``` 
    
3.  **Deploy Node.js Application:**

    
    ```bash
    kubectl apply -f nodejs-app-deployment.yaml
    ``` 
    
4.  **Create Node.js Application Service:**

    
    ```bash
    kubectl apply -f nodejs-app-service.yaml
    ``` 
    
5.  **Deploy Prometheus:**

    
    ```bash
    kubectl apply -f prometheus-config.yaml
    kubectl apply -f prometheus-deployment.yaml
    kubectl apply -f prometheus-service.yaml
    ``` 
    
6.  **Deploy Grafana:**

    
    ```bash
    kubectl apply -f grafana-deployment.yaml
    kubectl apply -f grafana-service.yaml
    ``` 
    
7.  **Create Ingress:**

    
    ```bash
    kubectl apply -f nodejs-app-ingress.yaml
    ``` 
    
8.  **Create Secrets:**

    
    ```bash
    kubectl apply -f db-secrets.yaml
    ``` 
    

## 4. Verify the Deployment

1.  **Check the Pods:**

    
    ```bash
    kubectl get pods
    ``` 
    
    Ensure that all pods are running.
    
2.  **Check the Services:**

    
    ```bash
    kubectl get services
    ``` 
    
    Verify that services for both Prometheus and Grafana are exposed.
    
3.  **Check the Ingress:**

    
    ```bash
    kubectl get ingress
    ``` 
    
    Ensure that the Ingress is set up correctly.
    

## 5. Accessing the Services

1.  **Node.js Application:**
    
    To access the Node.js application from your browser, add the Ingress hostname to your `/etc/hosts` file:

    
    ```bash
    echo "$(minikube ip) nodejs-app.local" | sudo tee -a /etc/hosts
    ``` 
    
    Then, open your browser and navigate to `http://nodejs-app.local`.
    
2.  **Grafana:**
    
    Access Grafana using the external IP address obtained from the service:

    
    ```bash
    kubectl get service grafana
    ``` 
    
    Open your browser and navigate to `http://<external-ip>:3000`.
    
3.  **Prometheus:**
    
    Access Prometheus using the external IP address obtained from the service:

    
    ```bash
    kubectl get service prometheus
    ``` 
    
    Open your browser and navigate to `http://<external-ip>:9090`.
    

## 6. Cleanup

To delete the deployment and associated resources:

1.  **Delete the Ingress:**

    
    ```bash
    kubectl delete -f nodejs-app-ingress.yaml
    ``` 
    
2.  **Delete the Node.js Application Service and Deployment:**

    
    ```bash
    kubectl delete -f nodejs-app-service.yaml
    kubectl delete -f nodejs-app-deployment.yaml
    ``` 
    
3.  **Delete the YugabyteDB Service and StatefulSet:**

    
    ```bash
    kubectl delete -f yugabyte-db-service.yaml
    kubectl delete -f yugabyte-db-statefulset.yaml
    ``` 
    
4.  **Delete Prometheus and Grafana:**

    
    ```bash
    kubectl delete -f prometheus-config.yaml
    kubectl delete -f prometheus-deployment.yaml
    kubectl delete -f prometheus-service.yaml
    kubectl delete -f grafana-deployment.yaml
    kubectl delete -f grafana-service.yaml
    ``` 
    
5.  **Delete Secrets:**

    
    ```bash
    kubectl delete -f db-secrets.yaml
    ```