# Blue-Green Deployment

**Blue — >Old**

**Green — >New** 

Build With Parameters —>Blue or Green

**Best Features:**

1.  Zero downtime 
2. Roll Back is very easy

### **Infrastructure:**

1. Create EC2 with t2.medium and 20GB Storage
2. Create a server for Jenkins with 25GB
3. Server for Sonar-Qube with 25GB
4. server for NEXUS with 25 GB

### SERVER(EC2):

**In-terminal:**

commands:

a . sudo apt-get update

b. connect to aws account

curl "[https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip](https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip)" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

c . aws configure

accesskey ,secretkey ,region and default output

d. Install Terraform

sudo snap install terraform —classic

e. clone the git hub url and add username and password (terrform eks cluster)

git clone [https://github.com/jaiswaladi246/Blue-Green-Deployment.git](https://github.com/jaiswaladi246/Blue-Green-Deployment.git)

- cd cluster
    - terrform init
    - terraform fmt
    - terraform plan
    - terraform apply

f. Install kubectl —> sudo snap install kubectl —classic

g . Connect to the cluster  —> 

aws eks —region ap-south-1 update-kubeconfig —name <clustername>

h . Kubectl get nodes

i.create a rbac service account role with name jenkins

sa.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
name: jenkins
namespace: webapps
```

j. kubectl create ns webapps

- kubectl apply -f sa.yaml

k. create a role

role.yaml

```yaml
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

kubectl apply -f role.yaml

l. Bind the role to service account

rolebind.yaml

```yaml
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

kubectl apply -f rolebinding.yaml

m. create a token 

kubectl create secret generic mysecretname \
--from-literal=[type=kubernetes.io/service-account-token](http://type=kubernetes.io/service-account-token) \
--dry-run=client -o yaml > secret.yaml

sec.yaml

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: jenkins

```

kubectl apply -f sec.yaml -n webapps

kubectl describe secret <name of secret> -n webapps

token generated and copy that token

### Jenkins Server:

1. sudo apt-get update
2. Install java

sudo apt install openjdk-17-jre-headless -y

1. install Jenkins

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
[https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key](https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key)
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
[https://pkg.jenkins.io/debian-stable](https://pkg.jenkins.io/debian-stable) binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins

1. sudo systemctl enable jenkins
2. sudo systemctl start jenkins
3. <public-ip> : 8080
4. paste admin password and install plugins and restart jenkins
5. Install Docker

`sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update`

1. sudo usermod -aG docker $USER
2. install trivy

sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - [https://aquasecurity.github.io/trivy-repo/deb/public.key](https://aquasecurity.github.io/trivy-repo/deb/public.key) | sudo apt-key add -
echo deb [https://aquasecurity.github.io/trivy-repo/deb](https://aquasecurity.github.io/trivy-repo/deb) $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

1. Install Kubectl

sudo snap install kubectl —classic

1. Credentials > Global >

Add Credentials > 

Kind: secret text

scope: Global

secret: paste the token

ID: k8-token

description: k8-token

1. **Install plugins:**
    - SonarQube scanner
    - Config File Provider
    - Maven Integration
    - Pipeline Maven Integration
    - pipe-line stage view
    - Docker pipeline
    - kubernetes (credentials,CLI,Client API)
    
    **Tools:** 
    
    - Add Maven
        - name: maven3
    - Sonarqube-scanner
        - name:sonarscanner
    
    **Credentials:**
    
    - Add sonar scanner credentials
        - Kind-secret text
        - scope-Global
        - Secret - {sonarsecret text}
        - Id : sonar-token
        - description- sonar-token
    
    **System:**
    
    - SonarQube installations
        - name: sonar
        - serverurl: <sonarqube publicip:9000>
        - server authentication token: sonar-token
    
    **Managed Files:**
    
    - Add new config
        - Global maven settings.xml
        - ID: maven-settings (next)
        - copy from nexus
        
        ```groovy
        <?xml version="1.0" encoding="UTF-8"?>
        
        <!--
        Licensed to the Apache Software Foundation (ASF) under one
        or more contributor license agreements.  See the NOTICE file
        distributed with this work for additional information
        regarding copyright ownership.  The ASF licenses this file
        to you under the Apache License, Version 2.0 (the
        "License"); you may not use this file except in compliance
        with the License.  You may obtain a copy of the License at
        
            http://www.apache.org/licenses/LICENSE-2.0
        
        Unless required by applicable law or agreed to in writing,
        software distributed under the License is distributed on an
        "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
        KIND, either express or implied.  See the License for the
        specific language governing permissions and limitations
        under the License.
        -->
        
        <!--
         | This is the configuration file for Maven. It can be specified at two levels:
         |
         |  1. User Level. This settings.xml file provides configuration for a single user, 
         |                 and is normally provided in ${user.home}/.m2/settings.xml.
         |
         |                 NOTE: This location can be overridden with the CLI option:
         |
         |                 -s /path/to/user/settings.xml
         |
         |  2. Global Level. This settings.xml file provides configuration for all Maven
         |                 users on a machine (assuming they're all using the same Maven
         |                 installation). It's normally provided in 
         |                 ${maven.home}/conf/settings.xml.
         |
         |                 NOTE: This location can be overridden with the CLI option:
         |
         |                 -gs /path/to/global/settings.xml
         |
         | The sections in this sample file are intended to give you a running start at
         | getting the most out of your Maven installation. Where appropriate, the default
         | values (values used when the setting is not specified) are provided.
         |
         |-->
        <settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" 
                  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
                  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
          <!-- localRepository
           | The path to the local repository maven will use to store artifacts.
           |
           | Default: ~/.m2/repository
          <localRepository>/path/to/local/repo</localRepository>
          -->
        
          <!-- interactiveMode
           | This will determine whether maven prompts you when it needs input. If set to false,
           | maven will use a sensible default value, perhaps based on some other setting, for
           | the parameter in question.
           |
           | Default: true
          <interactiveMode>true</interactiveMode>
          -->
        
          <!-- offline
           | Determines whether maven should attempt to connect to the network when executing a build.
           | This will have an effect on artifact downloads, artifact deployment, and others.
           |
           | Default: false
          <offline>false</offline>
          -->
        
          <!-- pluginGroups
           | This is a list of additional group identifiers that will be searched when resolving plugins by their prefix, i.e.
           | when invoking a command line like "mvn prefix:goal". Maven will automatically add the group identifiers
           | "org.apache.maven.plugins" and "org.codehaus.mojo" if these are not already contained in the list.
           |-->
          <pluginGroups>
            <!-- pluginGroup
             | Specifies a further group identifier to use for plugin lookup.
            <pluginGroup>com.your.plugins</pluginGroup>
            -->
          </pluginGroups>
        
          <!-- proxies
           | This is a list of proxies which can be used on this machine to connect to the network.
           | Unless otherwise specified (by system property or command-line switch), the first proxy
           | specification in this list marked as active will be used.
           |-->
          <proxies>
            <!-- proxy
             | Specification for one proxy, to be used in connecting to the network.
             |
            <proxy>
              <id>optional</id>
              <active>true</active>
              <protocol>http</protocol>
              <username>proxyuser</username>
              <password>proxypass</password>
              <host>proxy.host.net</host>
              <port>80</port>
              <nonProxyHosts>local.net|some.host.com</nonProxyHosts>
            </proxy>
            -->
          </proxies>
        
          <!-- servers
           | This is a list of authentication profiles, keyed by the server-id used within the system.
           | Authentication profiles can be used whenever maven must make a connection to a remote server.
           |-->
          <servers>
            <!-- server
             | Specifies the authentication information to use when connecting to a particular server, identified by
             | a unique name within the system (referred to by the 'id' attribute below).
             | 
             | NOTE: You should either specify username/password OR privateKey/passphrase, since these pairings are 
             |       used together.
          -->
            
        <server>
              <id>maven-release</id>
              <username>username of nexus</username>
              <password>password of nexus</password>
            </server>
            
        <server>
              <id>maven-snapshots</id>
              <username>username of nexus</username>
              <password>password of nexus</password>
            </server>
        
            
            <!-- Another sample, using keys to authenticate.
            <server>
              <id>siteServer</id>
              <privateKey>/path/to/private/key</privateKey>
              <passphrase>optional; leave empty if not used.</passphrase>
            </server>
            -->
          </servers>
        
          <!-- mirrors
           | This is a list of mirrors to be used in downloading artifacts from remote repositories.
           | 
           | It works like this: a POM may declare a repository to use in resolving certain artifacts.
           | However, this repository may have problems with heavy traffic at times, so people have mirrored
           | it to several places.
           |
           | That repository definition will have a unique id, so we can create a mirror reference for that
           | repository, to be used as an alternate download site. The mirror site will be the preferred 
           | server for that repository.
           |-->
          <mirrors>
            <!-- mirror
             | Specifies a repository mirror site to use instead of a given repository. The repository that
             | this mirror serves has an ID that matches the mirrorOf element of this mirror. IDs are used
             | for inheritance and direct lookup purposes, and must be unique across the set of mirrors.
             |
            <mirror>
              <id>mirrorId</id>
              <mirrorOf>repositoryId</mirrorOf>
              <name>Human Readable Name for this Mirror.</name>
              <url>http://my.repository.com/repo/path</url>
            </mirror>
             -->
          </mirrors>
          
          <!-- profiles
           | This is a list of profiles which can be activated in a variety of ways, and which can modify
           | the build process. Profiles provided in the settings.xml are intended to provide local machine-
           | specific paths and repository locations which allow the build to work in the local environment.
           |
           | For example, if you have an integration testing plugin - like cactus - that needs to know where
           | your Tomcat instance is installed, you can provide a variable here such that the variable is 
           | dereferenced during the build process to configure the cactus plugin.
           |
           | As noted above, profiles can be activated in a variety of ways. One way - the activeProfiles
           | section of this document (settings.xml) - will be discussed later. Another way essentially
           | relies on the detection of a system property, either matching a particular value for the property,
           | or merely testing its existence. Profiles can also be activated by JDK version prefix, where a 
           | value of '1.4' might activate a profile when the build is executed on a JDK version of '1.4.2_07'.
           | Finally, the list of active profiles can be specified directly from the command line.
           |
           | NOTE: For profiles defined in the settings.xml, you are restricted to specifying only artifact
           |       repositories, plugin repositories, and free-form properties to be used as configuration
           |       variables for plugins in the POM.
           |
           |-->
          <profiles>
            <!-- profile
             | Specifies a set of introductions to the build process, to be activated using one or more of the
             | mechanisms described above. For inheritance purposes, and to activate profiles via <activatedProfiles/>
             | or the command line, profiles have to have an ID that is unique.
             |
             | An encouraged best practice for profile identification is to use a consistent naming convention
             | for profiles, such as 'env-dev', 'env-test', 'env-production', 'user-jdcasey', 'user-brett', etc.
             | This will make it more intuitive to understand what the set of introduced profiles is attempting
             | to accomplish, particularly when you only have a list of profile id's for debug.
             |
             | This profile example uses the JDK version to trigger activation, and provides a JDK-specific repo.
            <profile>
              <id>jdk-1.4</id>
        
              <activation>
                <jdk>1.4</jdk>
              </activation>
        
              <repositories>
                <repository>
                  <id>jdk14</id>
                  <name>Repository for JDK 1.4 builds</name>
                  <url>http://www.myhost.com/maven/jdk14</url>
                  <layout>default</layout>
                  <snapshotPolicy>always</snapshotPolicy>
                </repository>
              </repositories>
            </profile>
            -->
        
            <!--
             | Here is another profile, activated by the system property 'target-env' with a value of 'dev',
             | which provides a specific path to the Tomcat instance. To use this, your plugin configuration
             | might hypothetically look like:
             |
             | ...
             | <plugin>
             |   <groupId>org.myco.myplugins</groupId>
             |   <artifactId>myplugin</artifactId>
             |   
             |   <configuration>
             |     <tomcatLocation>${tomcatPath}</tomcatLocation>
             |   </configuration>
             | </plugin>
             | ...
             |
             | NOTE: If you just wanted to inject this configuration whenever someone set 'target-env' to
             |       anything, you could just leave off the <value/> inside the activation-property.
             |
            <profile>
              <id>env-dev</id>
        
              <activation>
                <property>
                  <name>target-env</name>
                  <value>dev</value>
                </property>
              </activation>
        
              <properties>
                <tomcatPath>/path/to/tomcat/instance</tomcatPath>
              </properties>
            </profile>
            -->
          </profiles>
        
          <!-- activeProfiles
           | List of profiles that are active for all builds.
           |
          <activeProfiles>
            <activeProfile>alwaysActiveProfile</activeProfile>
            <activeProfile>anotherAlwaysActiveProfile</activeProfile>
          </activeProfiles>
          -->
        </settings>
        ```
        
        ```
        
        <server>
              <id>maven-release</id>
              <username>username of nexus</username>
              <password>password of nexus</password>
            </server>
            
        <server>
              <id>maven-snapshots</id>
              <username>username of nexus</username>
              <password>password of nexus</password>
            </server>
        ```
        
2. create a job Name : Blue-Green
- Discard Old builds - 2

**pipeline script:**

```groovy
pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }
    
    environment {
        IMAGE_NAME = "adijaiswal/bankapp"
        TAG = "${params.DOCKER_TAG}"  // The image tag now comes from the parameter
        KUBE_NAMESPACE = 'webapps'
        SCANNER_HOME= tool 'sonar-scanner'
    }

    stages {
      stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/jaiswaladi246/3-Tier-NodeJS-MySql-Docker.git'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('Tests') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        stage('Trivy FS scan') {
            steps {
                sh "trivy fs --format table -o fs.html . "
            }
        }

        
        stage('SonarQube Analysis') {
            steps {
                // sonar server name
                withSonarQubeEnv('sonar') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=Multitier -Dsonar.projectName=Multitier -Dsonar.java.binaries=target"
                }
            }
        }
        // In pipeline script search for timeout wait for Qualitygate
         stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit:'Hours'){
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        // withMaven: Provide Maven environment
         stage('Publish Artifact to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', jdk:'', maven:'maven3',mavenSettingConfig:'',traceability:true)
                    sh "mvn deploy -DskipTests=true"
            }
        }
        
        
        
        stage('Docker build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                    }
                }
            }
        }
        
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o image.html ${IMAGE_NAME}:${TAG}"
            }
        }
        
        stage('Docker Push Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }

        
        stage('Deploy MySQL Deployment and Service') {
            steps {
                // Here Server url is EKS API server endpoint
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://46743932FDE6B34C74566F392E30CABA.gr7.ap-south-1.eks.amazonaws.com') {
                        sh "kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}"  // Ensure you have the MySQL deployment YAML ready
                    }
                }
            }
        }
        
        stage('Deploy SVC-APP') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://46743932FDE6B34C74566F392E30CABA.gr7.ap-south-1.eks.amazonaws.com') {
                        sh """ if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                                kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                              fi
                        """
                   }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def deploymentFile = ""
                    if (params.DEPLOY_ENV == 'blue') {
                        deploymentFile = 'app-deployment-blue.yml'
                    } else {
                        deploymentFile = 'app-deployment-green.yml'
                    }

                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://46743932FDE6B34C74566F392E30CABA.gr7.ap-south-1.eks.amazonaws.com') {
                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }
        
        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV

                    // Always switch traffic based on DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://46743932FDE6B34C74566F392E30CABA.gr7.ap-south-1.eks.amazonaws.com') {
                        sh '''
                            kubectl patch service bankapp-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"bankapp\\", \\"version\\": \\"''' + newEnv + '''\\"}}}" -n ${KUBE_NAMESPACE}
                        '''
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://46743932FDE6B34C74566F392E30CABA.gr7.ap-south-1.eks.amazonaws.com') {
                        sh """
                        kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                        kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }
    }
}
```

For Gate Quality Check : In pipeline script search for timeout and wait for Qualitygate

In Deployment Sections we need to do —> 

1. My sql 
2. Service of bank-app

### Sonar-Qube Server:

1. sudo apt-get update
2. sudo apt install [docker.io](http://docker.io) -y
3. 
4.  (adding ubuntu user to docker group)
5. newgrp docker 
6. docker run -d -p 9000:9000 sonarqube:lts-community
7. docker ps -a

publicip:9000 —> to Login SonarQube server

*username:* admin

*password:* admin 

then change the password

docker run -d —name <name> -p <host-port>:<container port> imagename

for Credentials—>

Administration > security  > users > token(create a token and paste it)

Create a webhook —> 

Administration > Configuration > webhooks

create a webhook:

- Name: Jenkins
- URL: <jenkinsip:8080/sonarqube-webhook/

### Nexus Server:

1. sudo apt-get update
2. sudo apt install [docker.io](http://docker.io) -y
3. sudo usermod -aG docker ubuntu (adding ubuntu user to docker group)
4. newgrp docker 
5. docker run -d -p 8081:8081 sonatype/nexus3
6. docker ps -a 

*username:* admin

*password:* /nexus-data/admin.password

in terminal:

docker exec -it <container-id> /bin/bash

cd sonatype-work/nexus3

cat admin.password —> copy the password and paste it in nexus webpage

copy maven-releases url

copy maven-snapshots url

```groovy
<distributionManagement>
        <repository>
            <id>maven-releases</id>
            <url>http://65.2.129.62:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>maven-snapshots</id>
            <url>http://65.2.129.62:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
```

**pom.xml**

```groovy
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.3.3</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>bankapp</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>bankapp</name>
	<description>Banking Web Application</description>
	<url/>
	<licenses>
		<license/>
	</licenses>
	<developers>
		<developer/>
	</developers>
	<scm>
		<connection/>
		<developerConnection/>
		<tag/>
		<url/>
	</scm>
	<properties>
		<java.version>17</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-thymeleaf</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.thymeleaf.extras</groupId>
			<artifactId>thymeleaf-extras-springsecurity6</artifactId>
		</dependency>

		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>8.0.33</version>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.security</groupId>
			<artifactId>spring-security-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>

	
	</build>

		<distributionManagement>
        <repository>
            <id>maven-releases</id>
            <url>http://65.2.129.62:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>maven-snapshots</id>
            <url>http://65.2.129.62:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>

</project>
```