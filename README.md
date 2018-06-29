# jenkins-training

1. Start Jenkins: `docker run -d -p 8080:8080 --name jenkins jenkins/jenkins:lts`
2. Get the password: `docker logs jenkins`
3. Configure a Gradle Tool Install from gradle.org
4. Create the Free Style project:
   - GitHub Project: checked
   - Project url: `https://github.com/JetBrains/kotlin-examples/`
   - SCM Git:
      - Repo URL: `https://github.com/JetBrains/kotlin-examples.git`
   - Build 
      - Add Invoke Gradle Script
      - Gradle Version (Use selection created above)
      - Task: build run
      - Root Build Script: `${workspace}/gradle/hello-world/`
   - Post-build action:
      - Files to Archive: `gradle/hello-world/build/libs/*.jar`
5. Run the job.

Extra Credit: Can you create this job as a pipeline?
