# Complete-CI-CD
Creating a complete CI/CD corporate pipeline. Mimicking exactly what a large scale application would need.

## Step 1: Setup environment
### Setup AWS
Create Custom VPC
Create AWS security groups with ports open for SMTP, HTTP, SSH, and custom ports range for application
Create 3 EC2 instances for VMs (Master, Node 1, Node 2)

Use MobaXterm to verify ssh connection. (get public ip and use key pair)

### Setup K8
run ```sudo su``` to become root user
1. Update System Packages [On Master & Worker Node]

```sudo apt-get update```

2. Install Docker[On Master & Worker Node]

```sudo apt install docker.io -y```
```sudo chmod 666 /var/run/docker.sock```

3. Install Required Dependencies for Kubernetes[On Master & Worker Node]

```sudo apt-get install -y apt-transport-https ca-certificates curl gnupg```
```sudo mkdir -p -m 755 /etc/apt/keyrings```

4. Add Kubernetes Repository and GPG Key[On Master & Worker Node]

```curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg```

```echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list```

5. Update Package List[On Master & Worker Node]

```sudo apt update```

6. Install Kubernetes Components[On Master & Worker Node]

```sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1```

7. Initialize Kubernetes Master Node [On MasterNode]

```sudo kubeadm init --pod-network-cidr=10.244.0.0/16```
This command will create a subnetwork for our applications
It will give us a command to join the Nodes to the Master

8. Configure Kubernetes Cluster [On MasterNode]

```mkdir -p $HOME/.kube```
```sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config```
```sudo chown $(id -u):$(id -g) $HOME/.kube/config```

9. Deploy Networking Solution (Calico) [On MasterNode]

```kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml```

10. Deploy Ingress Controller (NGINX) [On MasterNode]

```kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml```

### Setup VMs for Sonarqube Nexus and Jenkins
#### AWS setup
Create 3 new AWS instances running ubuntu (The Jenkins one requires more RAM)
Using MobaXterm connect and check

#### Setting up Docker on SnoarQube and Nexus machines
Start with installing docker with below script

```
#!/bin/bash

# Update package manager repositories
sudo apt-get update

# Install necessary dependencies
sudo apt-get install -y ca-certificates curl

# Create directory for Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings

# Download Docker's GPG key
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

# Ensure proper permissions for the key
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository to Apt sources
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package manager repositories
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin 
```
The following command allows docker to be run by all users: ```sudo chmod 666 /var/run/docker.sock```

#### SonarQube Docker Container
```docker run -d --name sonar -p 9000:9000 sonarqube:lts-community```
First port is host port (port on machine)
Second port is container port. Container port 9000 communicates with machine port 9000
sonarqube:lts-community is a free sonarqube image to use
```docker ps`` to check

#### Nexus Docker Container
```docker run -d --name Nexus -p 8081:8081 sonatype/nexus3```
```docker exec -it CONTAINER ID /bin/bash```
it (interact terminal) to get nexus-password -> cd sonatype -> cd nexus3
Accessing the public IPs of these servers shows that I can access them both

#### Setting up Jenkins machine
```
#!/bin/bash

# Install OpenJDK 17 JRE Headless
sudo apt install openjdk-17-jre-headless -y

# Download Jenkins GPG key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins repository to package manager sources
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update package manager repositories
sudo apt-get update

# Install Jenkins
sudo apt-get install jenkins -y
```
Then install docker using same script as before and run chmod cmd for permission

## Step 2: Setup Github Repo and application
### Setup Github Repo
Create new repo for codebase
Fill codebase with app (boardgame in this instance) and push

## Step 3: CI/CD Pipline
### Jenkins
Plugins:
 - Eclipse Temurin (To handle multiple JDK version)
 - Config File Provider (used to create settings config files for initial settings) 
 - Pipeline Maven Integration (Allow maven use within pipeline connected to nexus)
 - Maven Integration
 - SonarQube Scanner (tool that performs analysis and publishes report to OUR server)
 - Docker & Docker pipeline
 - Kubernetes, Kubernetes CLI, Kubernetes Client API, Kubernetes Credentials

#### Configure Jenkins Plugins
Setup the plugins to install specific version of tools as listed below:
JDK - version = jdk17
SonarQube Scanner - verion = latest
Maven - maven3 - version = 3.6.1
Docker - version = latest

#### Create Pipeline
##### Jenkins Script
Create a pipeline to run for the boardgame
Start by defining tools.
Initialize git repo to pull code from.
Compile and run tests with maven shell script
##### Installing Trivy on Jenkins Server
Jenkins Server needs trivy installed CI scanning
```
#!/bin/bash

sudo apt-get install wget apt-transpoty-https gnupg lsb-release

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /uusr/share/keyrings/trivy.gpg > /dev/null

echo "deb [signed-by=usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/source.list.dtrivy.list

sudo apt-get update

sudo apt-get install trivy -y
```
##### Jenkins Script
Start Trivy file scanner
##### Connect SQ server to Jenkins
We need to connect sonarqube server with jenkins server so sq jobs can run there
Go to Manage Jenkins -> Credentials -> System -> Global Credentials -> Select Secret Text
Go to SQ Server -> Administration -> Security -> Users -> Tokens -> Generate a Token -> Paste to Jenkins
Manage Jenkins -> System -> SonarQube Server -> Add server using above auth token
Back to Jenkins Job
##### Jenkins Script
Add SQ env step using Server name from above line as parameter
##### Create Webhook
Create Quality Gate
Go to SQ server -> Admin -> Config -> Create Webhook
##### Jenkins Script
Add Quality Gate to Jenkins Script
Use Maven to Package project
##### Setup Publish to Nexus
Go to Nexus Server
Add Maven Releases and Maven snapshots URL to pom.xml under distributionManagement
Need to add nexus server credentials
Manage Jenkins -> Managed Files -> Global Maven
It will generate a settings.xml file, just go down to server, uncomment, and add server info
id: maven-releases, username: admin, password: xxx
Same with maven-snapshots
##### Jenkins Script
Add block to Jenkins script "mvn deploy"
This will sent artifacts to Nexus
##### Setup Docker Credentials
Add credential with dockerhub username and password
##### Jenkins Script
Add Docker build and tag script using pipeline syntax tool. Point to already existing dockerfile
Use trivy to scan docker image before pushing
Push Docker Image

##### Setup K8 Role Based Access Control
Create Service Account on Master Node
```kubectl create ns webapps```
svc.yaml:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```
```kubectl apply -f svc.yaml```
Create Master Role
role.yaml:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```
```kubectl apply -f role.yaml```
Bind Master Role to Service Account
bind.yaml:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins 
```
```kubectl apply -f bind.yaml```
To get Jenkins account to connect to K8 cluster we need auth token
sec.yaml:
```
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: jenkins
```
```kubectl apply -f sec.yaml -n webapps```
```kubectl describe secret mysecretname -n webapps```
Jenkins Dashboard -> Manage Jenkins -> Credentials -> Global -> Add secret text -> Paste Text from kubectl describe
Get Kubernetes config info:
```cd ~/.kube | cat config``` and copy server address for endpoint, cluster name

Install kubectl on Jenkins Server:
k.sh:
```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```
##### Jenkins Script
Pipeline syntax -> with Kubeconfig and add to Jenkins Script

##### Manage Mail Notifications
Go to Google account settings -> Security -> 2 step verification -> App passwords
Jenkins Dashboard -> Manage > System -> Extended Email notifiation
SMTP: smtp.gmail.com
smtp port: 465
Use SSL -> Creat Cred -> Username: Email address, Password: Generated App password
Do the same for Email notifiation
##### Jenkins Script
Add post block for email

## Step 4: Monitoring
Create a new AWS EC2 instance called Monitor
Install and setup/run Prometheus, Grafana, blackbox exporter
### Setup Prometheus and blackbox
Setup blackbox exporter by editin prometheus.yaml file
Edit the blackbox, make the targets prometheus.io and our targetr application address
### Setup Grafana
Connections -> Add data source -> prometheus
Import dashboard -> use the blackbox prometheus dashboard
### Monitor Jenkins
Install Prometheus-metrics plugin
Install and run prometheus node exporter on the Jenkins Server
Add Jenkins IP to promtheus.yaml as new job
Create new grafana dashboard for node exporter