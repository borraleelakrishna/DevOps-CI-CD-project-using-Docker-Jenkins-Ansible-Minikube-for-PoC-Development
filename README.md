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
On this server we need to install Docker, Kubectl and Minikube.
So like the previous step we will keep the commands in a shell script file and execute it to install the required tools

#!/bin/sh
#install docker
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

#install kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

Step 7: Setting up Jenkins Pipeline
First we need to create a project. So in the Jenkin’s panel in the browser, click New Item in the left menu


In the new screen, enter a name and select Pipeline from the displayed list and click Ok button.


In the next screen select “GitHub hook trigger for GITScm polling”. Then click Apply and Save button


Next we need to generate a token in Jenkin’s panel that will be used in the GitHub webhook for triggering Jenkin’s pipeline on code push. So click the admin name/icon in the top right corner and click Configure from left menu. In the new screen scroll to API Token section, click Add new Token and click Generate button. Copy and save this generated token somewhere in your notepad


Now head over to your GitHub repository containing application/code files. Then click on Settings and in the new window click Webhooks from the left menu and click Add Webhook button in the content pane


In the payload url, enter the public IP url of your Jenkins server and append it with /github-webhook/ . Note the trailing forward slash. In the Secret input text field paste the token copied from the Jenkin’s panel. Keep the rest of the fields in the form unchanged and hit Add Webhook


It will check the connectivity as shown in the above image as a grey circle icon besides the webhook url. You can hit refresh button of your browser, if it is taking time. That icon should turn into a green check mark


We should have the following content in our repo:

i) Dockerfile

FROM centos:latest
RUN cd /etc/yum.repos.d/
RUN sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
RUN sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
RUN yum install httpd wget zip unzip -y
ADD https://www.tooplate.com/zip-templates/2121_wave_cafe.zip /var/www/html
WORKDIR /var/www/html
RUN unzip -o 2121_wave_cafe.zip
RUN cp -r 2121_wave_cafe/* .
RUN rm -rf 2121_wave_cafe 2121_wave_cafe.zip
CMD ["/usr/sbin/httpd","-D","FOREGROUND"]
EXPOSE 80 30000
ii) Deployment.yml file

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myfirstdevopsappdeployment
spec:
  replicas: 5
  selector:
    matchLabels:
      name: myapp
  template:
    metadata:
      labels:
        name: myapp
    spec:
      containers:
        - name: myapp
          image: kubemubin/devops-project-one
          ports:
            - containerPort: 80
iii) Service.yml file

kind: Service
apiVersion: v1
metadata:
  name: myfirstdevopsservice
spec:
  selector:
    name: myapp
  ports:
    - protocol: TCP
      # Port accessible inside cluster
      port: 80
      # Port to forward to inside the pod
      targetPort: 80
      # Port accessible outside cluster
      nodePort: 30000
  type: NodePort
iv) Ansible playbook file

- hosts: all
  become: true
  become_user: root
  tasks:
    - name: delete old deployment
      command: kubectl delete -f /home/ubuntu/Deployment.yml --kubeconfig=/home/ubuntu/.kube/config
      ignore_errors: true
    - name: delete old service
      command: kubectl delete -f /home/ubuntu/Service.yml --kubeconfig=/home/ubuntu/.kube/config
      ignore_errors: true
    - name: create new deployment
      command: kubectl apply -f /home/ubuntu/Deployment.yml --kubeconfig=/home/ubuntu/.kube/config
    - name: create new service
      command: kubectl apply -f /home/ubuntu/Service.yml --force --kubeconfig=/home/ubuntu/.kube/config
Now back to the Jenkin’s panel, click on your project name and then click Configure from the left menu. In the content pane, scroll down to the Pipeline section, select “Pipeline Script” from the drop down. Below the textarea field there is a link to the Pipeline syntax. Click on it to setup certain global variables and credentials for our pipeline script. In the new window do the following:

a) Setup ansible-server and kubernetes-server SSH Agent variables.
Select sshagent option from the Sample Step dropdown and click Add button and click Jenkins icon from it.


In new pop-up window, select “SSH username with private key”. Then enter ID and description as ansible-server. Then set the username as “ubuntu”. Then select the Enter directly option in Private key and click add. In the resulting input field paste the content of .pem file which you downloaded while creating login key pair. Click Add.


Do the same for Kubernetes/Web server and generate kubernetes-server variable.

b) Generate variable for DockerHub automatic login from Ansible Server
Register on DockerHub then on the Pipeline Syntax page. Select withCredentials in the Sample Step dropdown field. Click Add button under Binding section and select “secret text” and enter variable name, like, “dockerhub_passwd”. Then under credentials sub-section, click add, click Jenkins icon. In the pop-up window, select secret text as kind and enter the ID same variable name as variable name you used. Click add to close pop-up window

Enter the following content in the Script textarea field

ansible_server_private_ip="<your-ansible-server-private-ip>"
kubernetes_server_private_ip="<your-web-server-private-ip>"

node{
    stage('Git checkout'){
        //replace with your github repo url
        git branch: 'main', url: 'https://github.com/khalifemubin/devops-project-one.git'
    }
    
     //all below sshagent variables created using Pipeline syntax
    stage('Sending Dockerfile to Ansible server'){
        sshagent(['ansible-server']) {
            sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip}"
            sh "scp /var/lib/jenkins/workspace/devops-project-one/* ubuntu@${ansible_server_private_ip}:/home/ubuntu"
        }
    }
    
    stage('Docker build image'){
        sshagent(['ansible-server']) {
         //building docker image starts
            sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} cd /home/ubuntu/"
            sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image build -t $JOB_NAME:v-$BUILD_ID ."
         //building docker image ends
         //Tagging docker image starts
            sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image tag $JOB_NAME:v-$BUILD_ID kubemubin/$JOB_NAME:v-$BUILD_ID"
         sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image tag $JOB_NAME:v-$BUILD_ID kubemubin/$JOB_NAME:latest"
         //Tagging docker image ends
        }
    }
    
    stage('push docker images to dockerhub'){
     sshagent(['ansible-server']) {
      withCredentials([string(credentialsId:'dockerhub_passwd', variable: 'dockerhub_passwd')]){
       sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker login -u kubemubin -p ${dockerhub_passwd}"
       sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image push kubemubin/$JOB_NAME:v-$BUILD_ID"
       sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image push kubemubin/$JOB_NAME:latest"
       
       //also delete old docker images
       sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} docker image rm kubemubin/$JOB_NAME:v-$BUILD_ID kubemubin/$JOB_NAME:latest $JOB_NAME:v-$BUILD_ID"
      }
        }
    }
    
    stage('Copy files from jenkins to kubernetes server'){
     sshagent(['kubernetes-server']) {
      sh "ssh -o StrictHostKeyChecking=no ubuntu@${kubernetes_server_private_ip} cd /home/ubuntu/"
      sh "scp /var/lib/jenkins/workspace/devops-project-one/* ubuntu@${kubernetes_server_private_ip}:/home/ubuntu"
     }
    }
 
    stage('Kubernetes deployment using ansible'){
     sshagent(['ansible-server']) {
      sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} cd /home/ubuntu/"
      sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} ansible -m ping ${kubernetes_server_private_ip}"
      sh "ssh -o StrictHostKeyChecking=no ubuntu@${ansible_server_private_ip} ansible-playbook ansible-playbook.yml"
     } 
    }
 
}
Let’s go through an overview of the pipeline script:

The first stage is to do git checkout of the repository. So whenever the changes are pushed to the main branch of the repo, this is the first thing pipeline will do.

Second step is to send everything (in the /var/lib/jenkins/workspace/<your-jenkins-projectname>/*) to the Ansible server at the home directory of ubuntu user.

Third stage is to build a Docker image with $JOB_NAME:v-$BUILD_ID format and tag the image.

Fourth stage is to push this built docker image to Dockerhub and delete the old image from the Ansible server.

Penultimate stage is to copy files from the checked out git repo to Kubernetes/Web server to the home directory of the ubuntu user.

Final stage is for Ansible server to run the playbook on Kubernetes/Web server.

Step 8: Start Minikube and See CI/CD in action
On the Kubernetes/Web server run minikube start . Since docker is already installed on this server it is similar to running minikube start — driver=docker . See the kubernetes cluster details kubectl get all . Now since we have nothing, lets add a dummy READMe.md file to our repo and push. Open up your project screen on Jenkin’s panel and you should see a the job getting built automatically


Now in the browser go to the public ip of web server and append node port to it: http://<public-ip-of-web-server>:30000

But there seems to be a problem. The web page is not rendering. Let’s do a small fix. Let’s forward the port of our service by running:

kubectl port-forward - address 0.0.0.0 svc/myfirstdevopsservice 30000:80 &
Et voilà !


If you found this post helpful, please like, share and follow me. I am a developer, transitioning to DevOps, not a writer—so each article is a big leap outside my comfort zone
