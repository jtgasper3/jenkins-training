# jenkins-training

## Starting a Jenkins instance
> Note: Running the Jenkins container running as the `root` user and linking to the host Docker environment (Docker in Docker) comes with inherent risk. There are other models for allowing Jenkins to manage Docker, but this method is done for expedience for the class.

1. Start Jenkins: `docker run -d -p 8080:8080 -v /var/run/docker.sock:/var/run/docker.sock -u root --name jenkins jenkins/jenkins:lts`
1. Get the password: `docker logs jenkins`
1. Finish the setup.

## First Free Style Project

1. Create the Free Style project:
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

Extra Credit: Can you create this job as a pipeline?

## Free Style Project with Docker

1. Install the `docker-build-step` plugin.
1. Setup the plugin to use the Host's Docker instance:
    - System Config -> Docker Builder
 	     - Docker URL: `unix:///var/run/docker.sock`
1. Create the Free Style project:
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
1. Check the console log.
   <details>
   <summary>Get the answer</summary>
   Jenkins created the Dockerfile and has started a Docker build using that file that installs <code>wget</code>.
   </details>


## Install Docker CLI into the Jenkins Container

> Note: For pipeline tasks run on the Jenkins master, the Docker CLI needs to be installed.

Execute the following commands, either using `docker exec -it jenkins bash` or create a one-time use Freestyle project with a Script task:

```
apt-get update
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
apt-get update
apt-get install docker-ce-cli
```

## Use a Pipeline Project to use Docker to Execute Tasks

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
   
   Jenkins has started a Docker container using the <code>node</code> image. It mounted the WORKSPACE directory into the container and is executing commands (`npm install`) inside the container.
   </details>

