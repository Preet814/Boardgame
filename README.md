
CI/CD Pipeline with Jenkins, SonarQube, Nexus, EKS, Prometheus & Grafana
This project demonstrates an end-to-end CI/CD pipeline on AWS using Jenkins, SonarQube, Nexus, Kubernetes (EKS), Trivy, Prometheus, and Grafana.

The pipeline ensures secure, automated, and monitored deployments into Kubernetes.

🚀 Architecture Overview
EC2 Instances

Jenkins Server (t3.medium) → CI/CD Orchestration

SonarQube Server (t3.medium) → Code Quality & Security Scanning

Nexus Server (t3.medium) → Artifact Repository (JAR/TAR storage)

Monitoring Server (t3.medium) → Prometheus & Grafana

AWS EKS Cluster

Managed Kubernetes control plane

Node Group with 2 EC2 worker nodes

Integrated with AWS ECR for container images

Monitoring

Prometheus + Grafana dashboards

Metrics scraped via node-exporter and kube-state-metrics

🔧 Step-by-Step Setup
1. Jenkins Setup (CI/CD Server)
Installed Java as a prerequisite.

Installed Jenkins and enabled the service.

Opened port 8080 in Security Group.

Installed necessary plugins:

Git, Pipeline, SonarQube Scanner, Docker, Kubernetes, AWS ECR, Email-ext Plugin, Trivy Integration.

Configured Jenkins credentials for GitHub, SonarQube, Nexus, AWS ECR, and SMTP for email notifications.

2. SonarQube Setup (EC2)
Installed SonarQube (default port 9000).

Created webhook → triggers back to Jenkins after Quality Gate check.

Pipeline halts if Quality Gate fails.

3. Nexus Setup (EC2)
Installed Nexus Repository Manager (port 8081).

Configured Maven repositories for JAR/TAR storage.

Linked Jenkins pipeline to upload built artifacts.

4. AWS EKS Cluster
Created EKS Cluster with IAM Role bindings.

Added Node Group with 2 worker nodes.

On Jenkins EC2:

Installed kubectl and AWS CLI.

Updated aws-auth ConfigMap with cluster role.

Verified worker nodes using:

bash
kubectl get nodes
5. CI/CD Pipeline (Jenkinsfile Stages)
Pipeline Flow:

Checkout Code → Clone from GitHub

SonarQube Analysis → Code quality and static analysis

Quality Gate Check → Pipeline halts if failed

Maven Build → mvn clean install (Package JAR)

Upload Artifact to Nexus → JAR pushed to Nexus repository

Docker Build → Create Image locally on Jenkins server

Trivy Security Scan (New Stage)

Run vulnerability scan on Docker image using Trivy

Generate vulnerability report in HTML/JSON format

Email report to developer/team

Pipeline halts if critical vulnerabilities found (configurable)

Docker Push to AWS ECR → Securely push scanned image

Kubernetes Deployment → Run kubectl apply -f deployment.yaml

Deploys 2 Pods into EKS

Configures NLB (Network Load Balancer) to expose service

Email Notification (Final)

When pipeline succeeds → Email summary sent to the team

6. Monitoring (Prometheus + Grafana)
Installed Prometheus and Grafana on Monitoring EC2.

From Jenkins EC2:

bash
kubectl apply -f node-exporter.yaml
kubectl apply -f kube-state-metrics.yaml
Prometheus scrapes Kubernetes cluster metrics.

Grafana visualizes performance dashboards for applications, pods, and nodes.

📊 Final Workflow
Developer pushes code → GitHub triggers Jenkins job

Jenkins pipeline runs:

Code analysis via SonarQube

Quality Gate → Fail/Halt or Continue

Build artifacts stored in Nexus

Docker image built → Trivy Scan run

Vulnerability report → Sent via Email

If clean → Push image to ECR

Deploy to EKS using Kubernetes manifests

Service exposed via NLB

Prometheus + Grafana → Monitor and visualize cluster/application health

Success Email → Sent at end of pipeline run

🖥️ Technologies Used
CI/CD: Jenkins

Code Quality: SonarQube

Artifact Repository: Nexus

Container Security: Trivy

Container Registry: AWS ECR

Orchestration: Kubernetes on AWS EKS

Monitoring & Visualization: Prometheus, Grafana

Cloud Infra: AWS EC2, IAM, NLB
