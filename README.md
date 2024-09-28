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

