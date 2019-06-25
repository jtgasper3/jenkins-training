# jenkins-training

Note: This lab should work with Docker on Linux, OS X, and Windows (with some additional steps). Jenkins does not work (well) with Play With Docker.

## Starting a Jenkins instance
> Note: Running the Jenkins container as the `root` user and linking to the host Docker environment (Docker in Docker) comes with inherent risk. There are other models for allowing Jenkins to manage Docker, but this method is done for expedience for the class.

1. Start Jenkins:
    - On Linux and OS X: `docker run -d -p 8080:8080 -v /var/run/docker.sock:/var/run/docker.sock -v $(which docker):/usr/local/bin/docker -u root --name jenkins jenkins/jenkins:lts`
    - On Windows: `docker run -d -p 8080:8080 -v /var/run/docker.sock:/var/run/docker.sock -u root --name jenkins jenkins/jenkins:lts`

1. Get the password: `docker logs jenkins`
1. Finish the setup.

## First Freestyle Project

1. Create the Freestyle project:
   - GitHub Project: checked
   - Project url: `https://github.com/JetBrains/kotlin-examples/`
   - SCM Git:
      - Repo URL: `https://github.com/JetBrains/kotlin-examples.git`
   - Build 
      - Add Invoke Gradle Script
      - User Gradle Wrapper
      - Wrapper Location: `${workspace}/gradle/hello-world/`
      - Task: `build run`
      - Root Build Script: `${workspace}/gradle/hello-world/`
   - Post-build action:
      - Files to Archive: `gradle/hello-world/build/libs/*.jar`
1. Save and run the job.
1. Is there a .jar file save with the job?

## Install Docker CLI into the Jenkins Container (Windows Only)

> Note: For pipeline jobs that use Docker and run on the Jenkins master, the Docker CLI needs to be installed on the master (which is a container. We mounted the host's `docker` command into the container for Linux and OS X, but it needs to be installed manually for Windows.

If you are running on Docker for Deskton Windows, execute the following commands, either using `docker exec -it jenkins bash` or create a one-time use Freestyle project with a Script task:

```
apt-get update -y \
&& apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    software-properties-common \
&& curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add - \
&& add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable" \
&& apt-get update -y \
&& apt-get install -y docker-ce-cli
```

## Freestyle Project with Docker (Plugins and System Configuration)

Let's make sure that Docker is working with our Jenkins install.

1. Install the `docker-build-step` plugin.
1. Setup the plugin to use the Host's Docker instance:
    - System Config -> Docker Builder
 	     - Docker URL: `unix:///var/run/docker.sock`
1. Create the Freestyle project:
1. Add a build step (`Execute shell`):
   - Execute command:
      - Script:
         ```
         cat > Dockerfile <<EOF
         FROM centos:7

         RUN yum update -y \
             && yum install -y wget

         EOF
         ```
1. Add a build step ('Execute Docker command'):
      - Docker command: `Create/build image`
      - Build context folder: `$WORKSPACE`
      - Tag of the resulting docker image: `test/test:$BUILD_NUMBER`
1. Save and run the job.
1. Check the console log, what do we expect will happen?
   <details>
   <summary>Get the answer</summary>
   
   Jenkins created the Dockerfile and has started a Docker build using the Dockerfile which installs `wget`.
   </details>

This demonstrates that our integration with the Docker CLI and the Docker engine are working. Normally, we would use a Dockerfile stored in a source code repo versus creating it with Jenkins.

## Use a Pipeline Job to build inside a Docker container

In this example, we use a Docker container not as an application host, but as the build tool.

1. Create a pipeline project.
1. Set the pipeline script:
   ```groovy
      pipeline {
       agent {
           docker {
               image 'node:6-alpine' 
               args '-p 3000:3000' 
           }
       }
       stages {
           stage('Build') { 
               steps {
                   sh 'npm install' 
               }
           }
       }
   }
   ```
1. Save and run the job.
1. Check the console log, what happened?
   <details>
   <summary>Get the answer</summary>
   
   Jenkins has started a Docker container using the `node` image. It mounted the WORKSPACE directory into the container and is executing commands (`npm install`) inside the container.
   </details>

## Docker Service Deployments 

### Discussion
Examine the following pipeline, and consider the following questions:

    - What does the following pipeline do?
    - What prerequisites need to happen before the pipeline would work?

```groovy
pipeline {
    agent {
        label 'docker'
    }
    stages {
        stage('Build') {
            steps {
                checkout([
                    $class: 'GitSCM', branches: [[name: '*/master']],
                    userRemoteConfigs: [[url: 'git@git.example.edu:example/app.git',credentialsId:'git-ssh-key']]
                ])
                sh 'docker image build --no-cache --pull --tag example/app:latest .'
            }
        }
        stage('Publish') {
            steps {
                sh 'docker image tag example/grouper:latest registry.example.edu/example/app:latest'
                sh "docker image tag example/grouper:latest registry.example.edu/example/app:${env.BUILD_NUMBER}"
                sh 'docker image push registry.example.edu/example/app:latest'
                sh "docker image push registry.example.edu/example/app:${env.BUILD_NUMBER}"
            }
        }
        stage('Run') {
            environment {
                DOCKER_TLS_VERIFY = 1
                DOCKER_HOST = 'tcp://iamswarm.example.edu:2376'
            }
            steps {
                withCredentials([dockerCert(credentialsId: 'iam-swarm', variable: 'DOCKER_CERT_PATH')]) {
                    sh "docker service update --with-registry-auth --image registry.example.edu/example/app:${env.BUILD_NUMBER} app"
               }
            }
        }
    }
}
```

   <details>
   <summary>Get the answer</summary>
   
   Starting from the top:
   1. The pipeline will only execute on a worker node that has a label `docker`.
   1. In the **Build** stage, we have to have a git/ssh credential setup, which checks out the source code on the worker node. Then the Docker image build occurs.
   1. In the **Publish** stage, we see that we are tagging and publishing the Docker image to a custom registry. The account hosting the Jenkins worker appears to already be logged in to the registry (since we don't see a credential block wrapping it).
   1. In the **Run** stage, we can see the connection to a Swarm Manager being configured. It will use TLS and we see that a Docker credential was previously setup to enable authentication to the Swarm Manager. We also see that a Docker Service, named `app` was previously `created` and we are applying a change to the service (changing the image tag to use the new version).
   
   Final note: the `docker service update` command, uses the `--with-registry-auth` parameter, which tells the Docker client to share its registry credential with the Docker Swarm so the Swarm can authenticate with the Registry and download the image.
   </details>

### Stack Example

Let's setup the Pets app from the Docker class so that Jenkins will deploy changes for us.

1. On your Docker host, enable Docker Swarm (`docker swarm init`).

2. Create a Pipeline project with the following pipeline code:

    ```groovy
    pipeline {
        agent any
        stages {
            stage('Checkout') {
                steps {
                    checkout([
                        $class: 'GitSCM', branches: [[name: '*/master']],
                        userRemoteConfigs: [[url: 'https://github.com/dockersamples/example-voting-app.git']]
                    ])
                }
            }
            stage('Run') {
                steps {
                    sh "docker stack deploy --with-registry-auth --compose-file docker-stack.yml vote"
                }
            }
        }
    }
    ```
    
3. Browse the [voting](http://localhost:5000) app and [results](http://localhost:5001) app. 

This is a good start. What improvements could we make it make the process better?

   <details>
   <summary>Get the answer</summary>
   
   - The Jenkins job must be manual triggers. We could setup SCM polling... Or, if we owned the GitHub repo, we could setup a Webhook to trigger the job for each new commit which is more efficient that polling.
   - The stack file has hard-coded image versions in it. Perhaps we could use environment variables to parameterize the stack file.
   - The application and image builds do not occur with Jenkins. That could be changed... If only the project had a Jenkinfile...
   </details>

## Bored? Done early?

Ideas:

    - Fork the Pets app project and try to improve it.
    - Review the list of plugins. Find anything interesting?

