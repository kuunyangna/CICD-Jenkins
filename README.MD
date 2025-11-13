# Jenkins CI/CD Pipeline with Docker Hub, Amazon EKS, Prometheus & Grafana

This project demonstrates a **complete DevOps automation pipeline** built with **Jenkins**, **Docker**, and **Amazon EKS**, with monitoring and observability provided by **Prometheus** and **Grafana**.

It covers containerization, continuous integration, continuous deployment, and cloud-native monitoring.

---

## Overview

The Jenkins pipeline automates the software delivery process from **code commit to production deployment**.

**Pipeline Stages:**

1. **Cleanup** – Cleans the Jenkins workspace to avoid conflicts.  
2. **Checkout Code** – Pulls the latest code from the GitHub repository.  
3. **Unit Testing** – Runs Maven unit tests.  
4. **Build Docker Image** – Builds the Docker image for the application.  
5. **Push to Docker Hub** – Pushes the Docker image to Docker Hub.  
6. **Deploy to EKS** – Updates kubeconfig, applies Kubernetes manifests, and deploys the app to Amazon EKS.  
7. **Monitoring Setup** – Deploys Prometheus and Grafana using Helm for observability.  

---

## Jenkins Credentials and Environment Variables

Configure the following **credentials in Jenkins**:

| Credential ID      | Purpose |
|-------------------|---------|
| `docker`          | Docker Hub username/password |
| `aws-creds`       | AWS IAM credentials with EKS permissions |

> The credentials are referenced in the pipeline using the `credentials()` function.

---

## Repository Structure

| File/Directory           | Description |
|--------------------------|-------------|
| `Jenkinsfile`            | Jenkins pipeline configuration |
| `Dockerfile`             | Docker build definition |
| `index.html`             | Web application HTML page |
| `k8-deployment.yaml`     | Kubernetes deployment manifest |
| `k8-service.yaml`        | Kubernetes service manifest |
| `pom.xml`                | Maven project file |
| `docs/architecture.png`  | Pipeline architectural diagram |

---

## Jenkins Pipeline Workflow

###1. Cleanup Stage
Removes all files from the Jenkins workspace to ensure a clean build.


```groovy
stage('Cleanup') {
    steps {
        cleanWs()
    }
}

###2. Checkout Code Stage
Pulls the latest version of the application code from the GitHub repository.
stage('Checkout Code') {
    steps {
        git branch: 'main', url: 'https://github.com/kuunyangna/CICD-Jenkins.git'
    }
}

## 3. Unit Testing Stage

Runs Maven unit tests to verify that the code functions as expected.

```groovy
stage('Unit Testing') {
    steps {
        sh 'mvn test'
    }
}


## 4. Build Docker Image Stage

Builds the Docker image locally from the Dockerfile in the project directory.

```groovy
stage('Build Docker Image') {
    steps {
        sh 'docker build -t myapp .'
    }
}


## 5. Push to Docker Hub Stage

Logs in to Docker Hub and pushes the built Docker image to your repository.

```groovy
stage('Build & Push Docker Image') {
    steps {
        sh '''
            echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin
            docker build -t kuunyangna/myapp-11:v2 .
            docker push kuunyangna/myapp-11:v2
        '''
    }
}

## 6. Deploy to EKS Stage

Updates kubeconfig and applies Kubernetes manifests to deploy the application on your EKS cluster.

```groovy
stage('Deploy to EKS') {
    steps {
        withCredentials([aws(credentialsId: 'aws-creds', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
            sh '''
                # Update kubeconfig to connect to your EKS cluster
                aws eks update-kubeconfig --region us-east-1 --name my-eks-cluster-final

                # Deploy application manifests
                kubectl apply -f k8-deployment.yaml
                kubectl apply -f k8-service.yaml

                # Check rollout status
                kubectl rollout status deployment/myapp-deployment
            '''
        }
    }
}


Monitoring Setup with Helm

1. Install Helm  
On the Jenkins host:  
```bash
sudo snap install helm --classic


2. Add Helm repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update


3. Deploy Prometheus & Grafana

```bash
# Create monitoring namespace if not exists
kubectl create namespace monitoring


# Deploy Prometheus and Grafana with gp2 storage class

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=gp2 \
  --set grafana.persistence.storageClassName=gp2


4. Access Grafana

Get the Grafana admin password:

```bash
kubectl get secret --namespace monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

Port-forward to local machine:

```bash
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring


5. Access Prometheus

Port-forward Prometheus:

```bash
kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090


Access Prometheus:  
http://localhost:9090

You can also use the LoadBalancer URL for Grafana and Prometheus to access them externally.


6. Import Dashboards in Grafana

Recommended dashboards for EKS and Kubernetes monitoring:

| Dashboard Purpose             | ID (Grafana) |
|-------------------------------|--------------|
| Node Exporter Metrics         | 1860         |
| Kubernetes Cluster Monitoring | 6417         |
| Kubernetes Metrics Overview   | 315          |
| Prometheus 2.0 Overview       | 3662         |


Steps to import:

Login to Grafana → + → Import  
Paste Dashboard ID  
Select Prometheus as the data source → Click Import

---

Author
Aquila Kuunyangna  
DevOps Engineer | 3x AWS Certified | Cloud & Automation Enthusiast  
[LinkedIn Profile](https://www.linkedin.com/in/kuunyangna)
