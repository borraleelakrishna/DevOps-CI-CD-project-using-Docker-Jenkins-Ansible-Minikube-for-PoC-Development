# DevOps-CI-CD-project-using-Docker-Jenkins-Ansible-Minikube-for-PoC-Development

Step 1: Install Jenkins
To install Jenkins perform the following steps in the SSH connected shell:

a) First, add the repository key to the system:
wget -q -O - http://pkg.jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -

b) Append the Debian package repository address to the server’s sources.list:
sudo sh -c 'echo deb http://pkg.jenkins-ci.org/debian binary/ > /etc/apt/sources.list.d/jenkins.list'

c) Run update so that the apt will use the new repository: sudo apt update

d) Install OpenJDK: sudo apt install openjdk-11-jdk

e) Finally, install Jenkins and its dependencies:
sudo apt install jenkins -y

and check it’s status after installation: systemctl status jenkins



step 2: Set Ansible Server

vi ansible-server-setup.sh

#!/bin/sh
#install ansible
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt update -y
sudo apt install ansible -y

#now install docker
sudo apt-get update
sudo apt-get install -y  \
    ca-certificates \
    curl \
    gnupg

sudo install -y -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo rm -rf /etc/docker/daemon.json
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -a -G docker ubuntu 
sudo systemctl start docker

chmod +x ansible-server-setup.sh
./ansible-server-setup.sh


Now configure the Web server as a host for Ansible. Edit /etc/ansible/hosts file and at the end the file, enter the following :
[webnode]
<private-ip-of-web-server>


Set Kubernetes/Web Server:

