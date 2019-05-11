# Project Demonstration


In this project, I will explain how to *use Git, Jenkins, Ansible, DockerHub, Docker to DEPLOY on a docker container.,*


#### PreRequisites
1. Jenkins 
2. Ansible 
3. Setup ansible client and install docker.
4. Docker Hub account 


In *part-01* I created Docker image on ansible server through Jenkins job and pushed it onto DockerHub. 

### Part-01 : Create an docker image 
    
1. Login to Jenkins console
1. Create *Jenkins job*, Fill the following details,
   - *Source Code Management:*
      - Repository : `https://github.com/sreenivasaredy7007/IBM.git`
      - Branches to build : `*/master`  
   - *Build:*
     - Root POM:`pom.xml`
     - Goals and options : `clean install package`
   - *Post Steps*
     - *Send files or execute commands over SSH*
       - Name: `ansible_server`
       - Source files	: `webapp/target/*.war`
       - Remove prefix	: `webapp/target`
       - Remote directory	: `//opt//docker`

     - *Send files or execute commands over SSH*
       - Name: `ansible_server`
       - Source files	: `Dockerfile`
       - Remote directory	: `//opt//docker`
       - Exec Command: 
	      - `cd /opt/docker`
          - `docker build -t project_demo .`
	      - `docker tag project_demo project/project_demo`
          - `docker push project/project_demo`
          - `docker rmi project_demo project/project_demo`
              

#### Troubleshooting:
1. Docker should be installed on ansible server 
2. Should login to "docker hub" on ansible server
3. ansadmin user should be part of `docker` group

In *Part-02* we create *create_docker_container.yml* playbook. this get intiated by jenkins job, run by ansible and exected on dokcer_host

### Part-02 : Deploy Containers

1. Write a yml file to create a container (file name : create_docker_container.yml)
   ```yaml
     ---
     - hosts: web-servers
       become: true
       tasks:
        - name: stop previous version docker
          shell: docker stop project_demo
        - name: remove stopped container
          shell: docker rm -f project_demo	  
        - name: remove docker images
          shell: docker image rm -f project/project_demo
          
        - name: create docker image
          shell: docker run -d --name project_demo -p 8090:8080 project/project_demo

1. Add this script to Jenkins job.
   - Chose *"configure"* to modify your jenkins job. 
     - *Under post build actions*
        - Send files or execute commands over SSH
          - Exec Command: 
          ```sh
             cd /opt/playbooks
             ansible-playbook create_docker_container.yml
            ```
          
1. Execute Jenkins job. 

1. You could see a new container on your docker host. can able access it from browser on port 8090

Troubleshooting: 
Makesure you have opened required ports on AWS Security group for this server. 

In *Part-03* we try to improvise to store docker images previous versions

### Part-03 : Deploy with Version Control Containers 

So for we used latest docker image to build a container, but what happens if latest version is not working?  
One easiest solution is, maintaining version for each build. This can be achieved by using environment variables. 

here we use 2 variables 
- `BUILD_ID` -  The current build id
- `JOB_NAME` - Name of the project of this build. This is the name you gave your job when you first set it up.


Lets modify jenkins job which was created in *Part-01* as below.

1. Create Jenkins job 
   - *Source Code Management:*
      - Repository : `https://github.com/sreenivasaredy7007/IBM.git`
      - Branches to build : `*/master`  
   - *Build:*
     - Root POM:`pom.xml`
     - Goals and options : `clean install package`
 
   - *Send files or execute commands over SSH*
     - Name: `ansible_server`
     - Source files	: `webapp/target/*.war`
     - Remove prefix	: `webapp/target`
     - Remote directory	: `//opt//docker`

   - *Send files or execute commands over SSH*
     - Name: `ansible_server`
     - Source files	: `Dockerfile`
     - Remote directory	: `//opt//docker`
      	- `cd /opt/docker`
        - `docker build -t $JOB_NAME:v1.$BUILD_ID .`
        - `docker tag $JOB_NAME:v1.$BUILD_ID project/$JOB_NAME:v1.$BUILD_ID`
        - `docker tag $JOB_NAME:v1.$BUILD_ID project/$JOB_NAME:latest`
        - `docker push project/$JOB_NAME:v1.$BUILD_ID`
        - `docker push project/$JOB_NAME:latest`
        - `docker rmi $JOB_NAME:v1.$BUILD_ID project/$JOB_NAME:v1.$BUILD_ID` 
        - `project/$JOB_NAME:latest`
