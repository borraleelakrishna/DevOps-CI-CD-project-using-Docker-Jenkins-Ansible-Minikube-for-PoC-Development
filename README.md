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
