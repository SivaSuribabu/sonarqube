# sonarqube
This Repo contains the sonarqube projects


SonarQube integration into Jenkins

Proper project structure in SonarQube UI

Reports separated like:

application-1
  ├── service1
  ├── service2
  ├── service3


I’ll explain this from installation → Jenkins integration → pipeline structure → UI organization strategy in a production-standard way.

1️⃣ SonarQube Installation (Production Approach)
Option A — Docker (Recommended for Dev / POC)
docker run -d --name sonarqube \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:lts


Access:

http://<server-ip>:9000


Default login:

admin / admin

Option B — Production Setup (Recommended)

Production should use:

External PostgreSQL

4GB+ RAM

Separate volume storage

Reverse proxy (Nginx)

HTTPS

Architecture:

Jenkins → SonarQube Server → PostgreSQL DB

2️⃣ Install SonarQube Plugin in Jenkins

Go to:

Manage Jenkins → Manage Plugins → Available


Install:

SonarQube Scanner


Then configure:

Manage Jenkins → Configure System


Add:

SonarQube Server

Name: sonarqube-prod

URL: http://<sonar-ip>:9000

Token: Generate from SonarQube

3️⃣ Create Token in SonarQube

Login →

My Account → Security → Generate Token


Copy token → Add in Jenkins as:

Manage Jenkins → Credentials → Add Secret Text

4️⃣ Important Question: Where to Integrate Sonar?

You have:

Stage1: Maven build
Stage2: Gradle build
Stage3: Gradle build
Final: ROOT.war

Correct Enterprise Approach:

✅ Run Sonar analysis for EACH SERVICE
❌ Don’t run only once at final WAR stage

Reason:

Sonar works at source-code level

Not at compiled WAR level

You want individual service quality gates

5️⃣ Recommended Sonar Structure in UI

You CANNOT create folder hierarchy manually like file system.

But you can organize using:

Project Keys

Portfolio (Enterprise edition)

Applications feature

Correct UI Structure (Community Edition Compatible)

Create separate projects:

application-1-service1
application-1-service2
application-1-service3


Then naming convention creates logical grouping.

If Using Developer Edition

You can create:

Applications → Add application

Application name:

application-1


Inside that, attach:

service1

service2

service3

That gives you exactly what you want in UI.

6️⃣ Pipeline Structure (Correct Implementation)

Now let’s modify your Jenkinsfile.

For Maven Service (Service1)
stage('Sonar - Service1') {
    steps {
        withSonarQubeEnv('sonarqube-prod') {
            sh """
            mvn clean verify sonar:sonar \
            -Dsonar.projectKey=application-1-service1 \
            -Dsonar.projectName=application-1-service1
            """
        }
    }
}

For Gradle Service (Service2 / Service3)

First add Sonar plugin in build.gradle:

plugins {
    id "org.sonarqube" version "4.4.1.3373"
}


Then pipeline stage:

stage('Sonar - Service2') {
    steps {
        withSonarQubeEnv('sonarqube-prod') {
            sh """
            ./gradlew sonarqube \
            -Dsonar.projectKey=application-1-service2 \
            -Dsonar.projectName=application-1-service2
            """
        }
    }
}

7️⃣ Add Quality Gate (Very Important)

After each sonar stage:

stage("Quality Gate") {
    steps {
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}


This ensures:

❌ If code fails quality gate
→ Pipeline fails
→ No WAR build

This is enterprise best practice.

8️⃣ Full Ideal Pipeline Flow
Stage 1: Build Service1
Stage 2: Sonar Service1
Stage 3: Quality Gate1

Stage 4: Build Service2
Stage 5: Sonar Service2
Stage 6: Quality Gate2

Stage 7: Build Service3
Stage 8: Sonar Service3
Stage 9: Quality Gate3

Stage 10: Build ROOT.war
Stage 11: Deploy

9️⃣ SonarQube UI Result

You’ll see:

Dashboard:

application-1-service1
application-1-service2
application-1-service3


If Developer edition:

Applications → application-1
→ Shows aggregated metrics
