# 1st_DevOps
this is use to pratice ansible, DevOps and other more for intern I4

# How to deply webapp using Jenkins, ansible and tomcat server
##1st version using ansible to deploy

//first create 3 server on VMware for

+ ansible server (AS)
+ Jenkins server (JS)
+ Tomcat server (TS)

/////////////////////////////////////////////////////////////////////

### ansible
- install ansible on our ansible server

- write playbook on our ansible server for automate install and configure on remote server

- use install_jenkins.yml to install jenkins for our jenkins server

- then use maven.yml to install apache-maven on jenkins server

- run playbook for install apache-tomcat to install tomcat on TS

### Jenkins 

- login jenkin server

    <JS_ip_address:8080(defult)>

- check password for login
     cat /var/lib/jenkins/secrets/initialAdminPassword

- go to .bachrc on JS to add path for JAVA and MAVEN
 
#nano .bashrc
  
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
M2_HOME=/opt/apache-maven-3.8.6

M2=$M2_HOME/bin

PATH=$PATH:$JAVA_HOME:$M2_HOME:$M2:$HOME/bin

#PATH=$PATH:$JAVA_HOME:$HOME/bin
export PATH

- source .bashrc  
- echo $PATH   

### tomcat
- go to TS and add some line on /opt/apache-tomcat-8.5.82/conf/tomcat-users.xml


 #ps -ef | grep tomcat-

  <role rolename="manager-gui"/>
  <role rolename="manager-script"/>
  <role rolename="manager-jmx"/>
  <role rolename="manager-status"/>
  <user username="admin" password="admin" roles="manager-gui,manager-script,manager-jmx,manager-status"/>
  <user username="deployer" password="deployer" roles="manager-script"/>
  <user username="tomcat" password="s3cret" roles="manager-gui"/>

- and go to 
  
       find / -name context.xml

       nano /opt/apache-tomcat/host-manager/META-INF/context.xml
       nano /opt/apache-tomcat/manager/META-INF/context.xml
  
  comment(#) on 

   <!--  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" /> -->

- go to /opt/apache-tomcat-8.5.82/bin/ 
  en add promission on startup.sh and shutdown.sh
 
      - chmod +x shutdown.sh
      - chmod +x startup.sh

- create shortcut for tomcat up and down
   
  - ln -s /opt/apache-tomcat-8.5.82/bin/startup.sh /usr/local/bin/tomcatup
  - ln -s /opt/apache-tomcat-8.5.82/bin/shutdown.sh /usr/local/bin/tomcatdown
   
- down and up tomcat server
   
  - tomcatdown
  - tomcatup

- open tomcat server on browser 

<TS_IP_ADDRESS:8080(defult)>

click manager app to login user:
  - username= tomcat
  - pwd: s3cret

//////////////////////////////////////////////////////////////////////////////////////
jenkins pipline script
//////////////////////////

    pipeline {
        agent any
    
    environment {
       
        registry = "theay003/mypythonapp"
        dockerImage = ''
        registryCredential = 'dockerhub_id'
    }
    stages {
        stage('checkout') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/AhTheary/python-basic']]])
            }
        }
        
        stage("Build Docker image") {
            steps {
                script{
                    dockerImage = docker.build registry
                }
            }
        }
        
        stage('uploading image') {
            steps{
                script {
                    docker.withRegistry( '', registryCredential ) {
                        dockerImage.push()
                    }
                }
            }
        }
            // Stopping Docker containers for cleaner Docker run
        stage('docker stop container') {
            steps {
                sh 'docker ps -f name=mypythonappContainer -q | xargs --no-run-if-empty docker container stop'
                sh 'docker container ls -a -fname=mypythonappContainer -q | xargs -r docker container rm'
            }
        }
        
        // Running Docker container, make sure port 8096 is opened in 
        stage('Docker Run') {
            steps{
                script {
                    dockerImage.run("-p 8096:5000 --rm --name mypythonappContainer")
                }
            }
        }
  

        }
    }
/////////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////


    pipeline {
        environment {
            registry = "theay003/devops"
            registryCredential = 'dockerhub_id'
            dockerImage = ''
        }

        agent any

        tools {
          nodejs 'node'
        }

        stages {

            stage('Cloning Git') {
                steps {
                    git 'https://github.com/AhTheary/Travel-Advisor.git'

                }
            }

            stage('Build') {
                steps {
                    sh 'npm install'
                }
            }

            stage('Test') {
                steps {
                    sh 'npm test'
                }
            }

            stage('Building image') {
                steps{
                    script {
                        dockerImage = docker.build registry + ":$BUILD_NUMBER"
                    }
                }
            }

            stage('Deploy Image') {
                steps{
                    script {
                        docker.withRegistry( '', registryCredential ) {
                            dockerImage.push()
                        }
                    }
                }
            }
            stage('docker stop container') {
            steps {
                sh 'docker ps -f name=mywebContainer -q | xargs --no-run-if-empty docker container stop'
                sh 'docker container ls -a -fname=mywebContainer -q | xargs -r docker container rm'
            }
        }

            // Running Docker container, make sure port 8096 is opened in 
            stage('Docker Run') {
                steps{
                    script {
                        dockerImage.run("-p 3000:3000 --rm --name mywebContainer")
                    }
                }
            }
        }
    }



