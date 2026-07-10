# End-to-End CI/CD Pipeline for Java Web Application

This project implements a complete CI/CD pipeline for a Java-based web application. The pipeline automates the build, containerization, artifact push, multi-environment testing, and Kubernetes deployment process using **Jenkins**, **Docker**, **Ansible**, and **Kubernetes (via kops)**.

## Architecture Overview

The setup uses the following servers:

| Server | Role |
|---|---|
| **Jenkins Server** | CI/CD orchestrator — builds artifact, creates Docker image, pushes to registry, triggers deployments |
| **Ansible Controller** | Manages QA server configuration and Docker container deployment |
| **QA Server 1 & QA Server 2** | Testing environments where the containerized app is deployed via Ansible |
| **Kubernetes Cluster (via kops)** | Production environment — 1 master node + 2 worker nodes |

---

## Complete Setup & Implementation Flow

### 1. Configure password-less SSH access

Password-less SSH was established from the Controller to QA Server1 and QA Server2, and from the Jenkins server to the Controller and the kops server. This allows Jenkins to trigger remote commands on these servers without manual authentication during pipeline execution.

### 2. Install Jenkins on the Jenkins server

Java and Jenkins were installed on an Ubuntu EC2 instance by following the official Jenkins installation guide ([jenkins.io](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)):

```bash
sudo apt-get update
```

The Java and Jenkins installation commands from the guide above were then run on the server. Once installed, Jenkins was accessed at:

```
http://<jenkins-server-public-ip>:8080
```

### 3. Unlock Jenkins and install plugins

The initial admin password was retrieved from the file path shown on the setup screen (opened via Git Bash or a similar tool) and entered to unlock Jenkins. All suggested plugins were then installed.

### 4. Create a new pipeline job

In the Jenkins dashboard: **New Item → name it "End to End Project" → select Pipeline → OK**.

### 5. Add the initial pipeline script and run the first build

The following pipeline script was added to pull the source code from GitHub and build it with Maven:

```groovy
pipeline {
    agent any
    stages {
        stage('Continuous Download') {
            steps {
                git 'https://github.com/kalyani1975018/maven.git'
            }
        }
        stage('Creation of Artifact') {
            steps {
                sh 'mvn package'
            }
        }
    }
}
```

After triggering the build, the console output was reviewed to confirm success and to identify the artifact's build location, for example:

```
Building war: /var/lib/jenkins/workspace/End to End Project/webapp/target/webapp.war
```

### 6. Install Docker on the Jenkins server and enable Docker access

Docker was installed on the Jenkins server, and the Jenkins user was added to the Docker group so Docker commands could run as part of the pipeline:

```bash
sudo usermod -aG docker jenkins
sudo su - jenkins
docker login
```

### 7. Create the Dockerfile

A Dockerfile was created to build a custom image on top of a TomEE base image, copying the built `.war` artifact into the TomEE webapps directory under a defined context path (`testapp`):

```dockerfile
FROM tomee
COPY webapp.war /usr/local/tomee/webapps/testapp.war
```

### 8. Automate artifact copy and Dockerfile creation within the pipeline

The pipeline was updated so that the `.war` file is copied from its build location into the workspace, and the Dockerfile is generated automatically via a shell step — removing the need for any manual copying:

```groovy
stage('Build Docker Image') {
    steps {
        sh 'cp webapp/target/webapp.war .'
        sh '''cat>Dockerfile<<EOF
FROM tomee
COPY webapp.war /usr/local/tomee/webapps/testapp.war
EOF
'''
        sh 'docker build -t kalyani91/javaapp .'
    }
}
```

### 9. Push the Docker image to Docker Hub

```groovy
stage('Push Image to Registry') {
    steps {
        sh 'docker push kalyani91/javaapp'
    }
}
```

The push was verified by checking the image listing on Docker Hub.

### 10. Configure Ansible to prepare the QA servers

On the Ansible controller, a playbook was created to install Docker, Python3-pip, and the Docker Python SDK on all target QA servers, so Ansible could communicate with Docker on those hosts:

```yaml
---
- name: Install docker and required software for ansible integration
  hosts: all
  become: yes

  tasks:
    - name: Install pip3
      apt:
        name: python3-pip
        state: present
        update_cache: yes

    - name: Download Docker installation script
      shell: curl -fsSL https://get.docker.com -o install-docker.sh

    - name: Install Docker
      shell: sh install-docker.sh

    - name: Install Docker Python SDK
      apt:
        name: python3-docker
        state: present

    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: yes
```

Run with:
```bash
ansible-playbook ansible_docker.yaml -b
```

### 11. Deploy the application container to the QA servers via Ansible

```yaml
---
- name: Download customized docker image and create container on all the QA servers
  hosts: all
  tasks:
    - name: Create container from the customized image
      docker_container:
        name: myapp
        image: kalyani91/javaapp
        ports:
          - 8888:8080
```

Run with:
```bash
ansible-playbook javaapp.yaml -b
```

This step was then added into the Jenkins pipeline itself, so the deployment to QA runs automatically after the image push:

```groovy
stage('Deploy the Docker Image into QA Servers') {
    steps {
        sh 'ssh ubuntu@172.31.47.247 ansible-playbook /home/ubuntu/javaapp.yaml -b'
    }
}
```

The application was then verified at `http://<qa-server-public-ip>:8888/testapp`.

### 12. Add a functional testing stage

```groovy
stage('Functional Testing') {
    steps {
        git 'https://github.com/kalyani1975018/FunctionalTesting.git'
        sh 'java -jar /var/lib/jenkins/workspace/End\\ to\\ End\\ Project/testing.jar'
    }
}
```

### 13. Provision a server for kops

An Amazon Linux EC2 instance was set up to run kops for Kubernetes cluster management.

### 14. Create and attach an IAM role for kops

Since kops needs to provision a VPC, Auto Scaling Groups, and other AWS resources, an IAM role was created and attached to the kops instance:

- **IAM → Roles → Create Role → AWS Service → EC2 → Permissions: AdministratorAccess → Role name: `DevopsProject_role`**
- The role was then attached to the kops EC2 instance via **Actions → Security → Modify IAM Role**.

### 15. Install kops and kubectl on the kops server

```bash
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops

curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

### 16. Create supporting AWS resources for the cluster

An S3 bucket was created to serve as the kops state store:

```bash
aws s3 mb s3://devopsprojecta.in.k8s --region us-west-2
```

A Route53 hosted zone was created for cluster DNS (name: `kalyani.in`, region: Oregon, default VPC selected). Since this configuration is only picked up after a fresh login, it was reloaded manually instead:

```bash
source .bashrc
ssh-keygen
```

### 17. Create the Kubernetes cluster using kops

```bash
kops create cluster \
  --state=${KOPS_STATE_STORE} \
  --node-count=2 \
  --master-size=t3.micro \
  --node-size=t3.micro \
  --zones=us-east-2a \
  --name=${KOPS_CLUSTER_NAME} \
  --dns private \
  --master-count 1
```

This provisioned **1 master node** and **2 worker nodes**.

### 18. Create the Kubernetes Deployment and Service manifests

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-deployment
  labels:
    name: java-app
    author: Kalyani
spec:
  replicas: 2
  selector:
    matchLabels:
      name: java-app
  template:
    metadata:
      name: java-app-pod
      labels:
        name: java-app
    spec:
      containers:
        - name: my-java-app
          image: kalyani91/javaapp
---
apiVersion: v1
kind: Service
metadata:
  name: java-app-service
  labels:
    name: java-app
spec:
  type: NodePort
  ports:
    - targetPort: 8080
      port: 8080
      nodePort: 30008
  selector:
    name: java-app
```

### 19. Automate the Kubernetes deployment within the pipeline

A final stage was added to the pipeline to connect to the kops server over SSH and apply the deployment manifest, completing the fully automated flow from source code commit to production deployment:

```groovy
stage('Deploy into K8s Cluster - Production') {
    steps {
        sh 'ssh ec2-user@172.31.34.185 kubectl apply -f javaapp.yaml'
    }
}
```

Deployment was verified using:
```bash
kubectl get pods
kubectl get svc
```

With 2 replicas defined in the manifest, 2 running pods were confirmed on the cluster.

---

## Complete Declarative Jenkins Pipeline

```groovy
pipeline {
    agent any
    stages {
        stage('Continuous Download') {
            steps {
                git 'https://github.com/kalyani1975018/maven.git'
            }
        }
        stage('Creation of Artifact') {
            steps {
                sh 'mvn package'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'cp webapp/target/webapp.war .'
                sh '''cat>Dockerfile<<EOF
FROM tomee
COPY webapp.war /usr/local/tomee/webapps/testapp.war
EOF
'''
                sh 'docker build -t kalyani91/javaapp .'
            }
        }
        stage('Push Image to Registry') {
            steps {
                sh 'docker push kalyani91/javaapp'
            }
        }
        stage('Deploy the Docker Image into QA Servers') {
            steps {
                sh 'ssh ubuntu@172.31.47.247 ansible-playbook /home/ubuntu/javaapp.yaml -b'
            }
        }
        stage('Functional Testing') {
            steps {
                git 'https://github.com/kalyani1975018/FunctionalTesting.git'
                sh 'java -jar /var/lib/jenkins/workspace/End\\ to\\ End\\ Project/testing.jar'
            }
        }
        stage('Deploy into K8s Cluster - Production') {
            steps {
                sh 'ssh ec2-user@172.31.34.185 kubectl apply -f javaapp.yaml'
            }
        }
    }
}
```

---

## Tech Stack

- **CI/CD:** Jenkins (Declarative Pipeline)
- **Build Tool:** Maven
- **Containerization:** Docker, TomEE base image
- **Configuration Management:** Ansible
- **Orchestration:** Kubernetes (provisioned with kops on AWS)
- **Cloud:** AWS (EC2, S3, Route53, IAM)
- **Source Control:** GitHub

Project is deployed successfully.
<img width="1920" height="1080" alt="Screenshot (155)" src="https://github.com/user-attachments/assets/26d28321-3173-4e1b-8194-ffa7bdefb1a6" />
<img width="1920" height="1080" alt="Screenshot (162)" src="https://github.com/user-attachments/assets/c8cc4eb8-4fd8-46fc-9aee-93f620e9255f" />
<img width="1920" height="1080" alt="Screenshot (167)" src="https://github.com/user-attachments/assets/c7e7704e-8698-4405-9e05-fb480e2bce76" />
<img width="1920" height="1080" alt="Screenshot (168)" src="https://github.com/user-attachments/assets/6d962573-1da0-458c-aba1-38cdfc260b3d" />
<img width="1920" height="1080" alt="Screenshot (169)" src="https://github.com/user-attachments/assets/c315c77b-cb66-4e69-a974-665a473de30c" />
<img width="1920" height="1080" alt="Screenshot (170)" src="https://github.com/user-attachments/assets/b9bfa412-cf11-45f9-b651-6e43e2741647" />
<img width="1920" height="1080" alt="Screenshot (171)" src="https://github.com/user-attachments/assets/61b0a651-6119-4228-aa63-53ce9029e609" />
<img width="1920" height="396" alt="Screenshot (150)" src="https://github.com/user-attachments/assets/242226e3-b72d-4dc9-92cc-7b0915b17a94" />
<img width="1920" height="1080" alt="Screenshot (159)" src="https://github.com/user-attachments/assets/8c732359-6c06-4cde-8ebc-39c7edbb32aa" />
<img width="1920" height="1080" alt="Screenshot (153)" src="https://github.com/user-attachments/assets/47acbe8b-4e11-4c53-930e-2caa324925af" />
<img width="1920" height="1080" alt="Screenshot (154)" src="https://github.com/user-attachments/assets/d6d9dee1-2c05-4160-ab94-46de3e6da60b" />




