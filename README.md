# Course: Building a Basic CI/CD Pipeline with Jenkins, Docker, and GitHub

## Overall Objective
After completing this course, you will be able to:
- Understand the fundamental concepts of CI/CD.
- Install and configure Jenkins, Docker, and Docker Compose.
- Write a Dockerfile to containerize an application (e.g., Spring Boot).
- Use Docker Compose to manage multi-container environments (application + database).
- Write a Jenkinsfile (Declarative Pipeline) to automate the build, basic test, Docker image packaging, and application deployment processes.
- Configure a GitHub Webhook to automatically trigger the Jenkins pipeline when code changes are pushed to GitHub.
- Successfully build and run a complete demo project with an automated CI/CD pipeline.

## Target Audience
- Programmers and beginner DevOps engineers who want to learn about CI/CD and related tools.

## Prerequisites
- Basic knowledge of Git and GitHub.
- Basic command-line knowledge (Linux/macOS/Windows PowerShell).
- A computer with an internet connection and sufficient resources to run Jenkins and Docker (recommended: at least 8GB RAM).
- (Optional) Basic knowledge of Java and Spring Boot (if you want to customize the demo application).

## Demo Project
A simple Spring Boot Web API application, initially without a complex database, to focus on the CI/CD pipeline. It can later be extended with a database using Docker Compose.

## Module 1: Introduction and Environment Setup

### 1.1 Objectives
- Understand the concept and benefits of CI/CD.
- Understand the roles of Jenkins, Docker, Docker Compose, and GitHub in the process.
- Successfully install Jenkins and Docker on your machine.

### 1.2 Brief Theory
- **CI (Continuous Integration)**: The practice of automating the integration of code changes from multiple contributors into a shared repository. Each integration is typically verified by automated builds and tests to detect errors early.
- **CD (Continuous Delivery/Deployment)**:
  - **Continuous Delivery**: Extends CI by ensuring that after successful builds and tests, the software is always ready to be deployed to production at any time. The final deployment is usually manual (e.g., pressing a button).
  - **Continuous Deployment**: Further automates the deployment to production if all prior stages (build, test, etc.) succeed.
- **Jenkins**: An open-source automation server that automates parts of the software development process, such as building, testing, and deploying. Jenkins Pipeline allows defining the entire process as code (Jenkinsfile).
- **Docker**: A containerization platform that packages an application and its dependencies into a standardized unit called a container, ensuring consistent execution across environments.
- **Docker Compose**: A tool for defining and running multi-container Docker applications using a YAML file to configure services, networks, and volumes.
- **GitHub**: A Git-based platform for source code management and collaboration.
- **GitHub Webhook**: A mechanism that allows GitHub to send notifications (HTTP POST) to an external URL (e.g., Jenkins) when events occur in the repository (e.g., code push).

### 1.3 Setup
**Important**: The easiest and recommended way to run Jenkins for learning purposes is using Docker.

#### Install Docker and Docker Compose
- Visit the Docker official site: [https://docs.docker.com/get-docker/](https://docs.docker.com/get-docker/)
- Select your operating system (Windows, macOS, Linux) and follow the instructions to install Docker Desktop (Windows/macOS) or Docker Engine + Docker Compose (Linux). Docker Desktop includes Docker Compose.
- Verify installation:
  ```bash
  docker --version
  docker-compose --version  # Or `docker compose version` (for newer versions)
  ```
  If versions are displayed, the installation is successful.

#### Run Jenkins with Docker
Open a terminal and run:
```bash
# Create a Docker bridge network (for Jenkins and other containers to communicate)
docker network create jenkins-net

# Run Jenkins controller (master) container
# Using image jenkins/jenkins:lts-jdk17 (LTS with JDK 17)
# -d: run in the background
# --name jenkins: name the container
# --network jenkins-net: connect to the created network
# -p 8080:8080: map port 8080 of the host to 8080 of the container
# -p 50000:50000: map port 50000 for agents (if needed)
# -v jenkins_home:/var/jenkins_home: persist Jenkins data outside the container
# -v /var/run/docker.sock:/var/run/docker.sock: allow Jenkins to control the host Docker daemon (IMPORTANT for building Docker images)
docker run -d --name jenkins --network jenkins-net \
-p 8080:8080 -p 50000:50000 \
-v jenkins_home:/var/jenkins_home \
-v /var/run/docker.sock:/var/run/docker.sock \
jenkins/jenkins:lts-jdk17
```

#### Retrieve Initial Admin Password
Wait about 1 minute for Jenkins to start, then run:
```bash
docker logs jenkins
```
Look for the admin password in the logs (between `******`). Alternatively:
```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

#### Access Jenkins
- Open a browser and go to [http://localhost:8080](http://localhost:8080).
- Paste the admin password into the provided field.
- Install Plugins: Select "Install suggested plugins" (this may take a few minutes).
- Create Admin User: Set up your first admin account.
- Instance Configuration: Confirm the Jenkins URL (usually [http://localhost:8080](http://localhost:8080)).
- Jenkins is now ready!

### 1.4 Practice
- Ensure you can access the Jenkins UI at [http://localhost:8080](http://localhost:8080).
- Verify that `docker --version` and `docker-compose --version` run successfully.

### 1.5 Testing
- Log in to Jenkins with the account you created.
- Try running a simple container:
  ```bash
  docker run hello-world
  ```

## Module 2: Containerizing a Spring Boot Application with Docker

### 2.1 Objectives
- Understand the basic structure of a Dockerfile.
- Write a Dockerfile to build and run a Spring Boot application.
- Build a Docker image from the Dockerfile.
- Run the Spring Boot application as a Docker container.

### 2.2 Brief Theory
- **Dockerfile**: A text file containing instructions for Docker to automatically build an image. Each instruction creates a layer in the image.
- Common Dockerfile instructions:
  - `FROM`: Specifies the base image (e.g., `openjdk:17-jdk-slim`).
  - `WORKDIR`: Sets the working directory for subsequent instructions (`COPY`, `RUN`, `CMD`, etc.).
  - `COPY` or `ADD`: Copies files/directories from the host to the image’s filesystem.
  - `RUN`: Executes a command during the image build (e.g., installing packages, compiling code).
  - `EXPOSE`: Declares the port the container will listen on (does not automatically map ports).
  - `CMD` or `ENTRYPOINT`: Specifies the default command to run when the container starts. `CMD` can be overridden with `docker run`, while `ENTRYPOINT` is harder to override.

### 2.3 Prepare the Demo Application
#### Option 1: Use Spring Initializr
- Visit [https://start.spring.io/](https://start.spring.io/)
- Select:
  - Project: Maven Project
  - Language: Java
  - Spring Boot version (e.g., 3.x.x)
  - Group: `com.example`
  - Artifact: `demo-cicd`
  - Dependencies: Add `Spring Web`
- Click "Generate," download the zip file, and extract it.

#### Option 2: Create Files Manually
Create the directory structure:
```
demo-cicd/
├── src/main/java/com/example/democicd/
│   ├── DemoCicdApplication.java
│   ├── HelloController.java
├── pom.xml
```

**pom.xml** (Minimal for Spring Boot Web):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.4</version>
        <relativePath/>
    </parent>
    <groupId>com.example</groupId>
    <artifactId>demo-cicd</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo-cicd</name>
    <description>Demo project for Spring Boot CI/CD</description>
    <properties>
        <java.version>17</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
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
</project>
```

**DemoCicdApplication.java**:
```java
package com.example.democicd;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoCicdApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoCicdApplication.class, args);
    }
}
```

**HelloController.java**:
```java
package com.example.democicd;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/")
    public String hello() {
        return "Hello from CI/CD Demo!";
    }
}
```

**Build the Application**:
In the `demo-cicd` directory, run:
```bash
./mvnw clean package  # Linux/macOS
# or
mvnw.cmd clean package  # Windows
```
This downloads dependencies and creates `target/demo-cicd-0.0.1-SNAPSHOT.jar`.

### 2.4 Write Code (Dockerfile)
In the root `demo-cicd` directory, create a file named `Dockerfile` (no extension) with:
```dockerfile
# Use OpenJDK 17 base image (slim version for smaller size)
FROM openjdk:17-jdk-slim

# Set working directory inside the container
WORKDIR /app

# Copy the built jar file from the host's target directory to /app in the container
# Note: The jar file name must match the one generated by `mvn package`
COPY target/demo-cicd-0.0.1-SNAPSHOT.jar app.jar

# Declare the port the Spring Boot application will run on (default: 8080)
EXPOSE 8080

# Command to run the application when the container starts
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 2.5 Practice & Testing
#### Build Docker Image
- Ensure you’ve run `./mvnw clean package` to generate the `.jar` file in the `target` directory.
- In the `demo-cicd` directory, run:
  ```bash
  # The . specifies the build context as the current directory
  docker build -t demo-cicd-app:latest .
  ```
- Verify the image was created:
  ```bash
  docker images
  ```
  You should see `demo-cicd-app` with the `latest` tag.

#### Run Container
Run the container, mapping host port 8081 to container port 8080:
```bash
# -d: run in the background
# -p 8081:8080: map host:container ports
# --name my-spring-app: name the container
# demo-cicd-app:latest: image name and tag
docker run -d -p 8081:8080 --name my-spring-app demo-cicd-app:latest
```

- Check running containers:
  ```bash
  docker ps
  ```
  You should see `my-spring-app`.
- View container logs:
  ```bash
  docker logs my spring-app
  ```
  Look for Spring Boot startup logs.
- Test: Open a browser and visit [http://localhost:8081](http://localhost:8081). You should see "Hello from CI/CD Demo!".
- Cleanup:
  ```bash
  docker stop my-spring-app
  docker rm my-spring-app
  ```

## Module 3: Using Docker Compose

### 3.1 Objectives
- Understand the basic syntax of `docker-compose.yml`.
- Define and run a Spring Boot application with a database (e.g., PostgreSQL) using Docker Compose.
- Connect the Spring Boot application to the database in a Docker Compose environment.

### 3.2 Brief Theory
- **docker-compose.yml**: A YAML configuration file defining services (containers), networks (virtual networks for service communication), and volumes (persistent data storage).
- **Services**: Each service corresponds to a container, built from an image (via Dockerfile or pulled from a registry). Configurable with ports, environment variables, networks, volumes, etc.
- **Networks**: Docker Compose creates a default network for services in the same `docker-compose.yml`, allowing communication by service name.
- **Volumes**: Used to persist container data (e.g., database data) outside the container’s filesystem, ensuring data is not lost when containers are removed and recreated.

### 3.3 Update Demo Application (Add Database)
#### Update `pom.xml`
Add dependencies for Spring Data JPA and PostgreSQL driver:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

#### Update `src/main/resources/application.properties`
Create or update the file with database connection settings. The `db` host will be resolved by Docker Compose to the database container’s IP:
```properties
# DataSource Configuration
spring.datasource.url=jdbc:postgresql://db:5432/mydatabase
spring.datasource.username=myuser
spring.datasource.password=mypassword
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA Configuration (optional, e.g., auto-create tables)
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```
Note: The username and password will be defined in `docker-compose.yml`.

#### Rebuild Application
```bash
./mvnw clean package
```

#### Rebuild Docker Image
Since `pom.xml` changed, rebuild the image to include the new `.jar`:
```bash
docker build -t demo-cicd-app:latest .
```

### 3.4 Write Code (docker-compose.yml)
In the root `demo-cicd` directory, create `docker-compose.yml`:
```yaml
version: '3.8' # Use a compatible compose version

services:
  # Service for the Spring Boot application
  app:
    image: demo-cicd-app:latest # Use the image built in Module 2
    container_name: spring-boot-app-compose
    ports:
      - "8081:8080" # Map host port 8081 to container port 8080
    environment:
      # Pass environment variables if needed, e.g., to override application.properties
      # SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/mydatabase
      # SPRING_DATASOURCE_USERNAME: myuser
      # SPRING_DATASOURCE_PASSWORD: mypassword
      SPRING_PROFILES_ACTIVE: default # Example: set active profile
    depends_on:
      - db # Ensure 'db' service starts before 'app'
    networks:
      - app-network # Connect to the shared network

  # Service for PostgreSQL database
  db:
    image: postgres:15 # Use the official PostgreSQL image
    container_name: postgres-db-compose
    environment:
      POSTGRES_DB: mydatabase # Name of the database to create
      POSTGRES_USER: myuser # Username for connection
      POSTGRES_PASSWORD: mypassword # Password for the user
    volumes:
      - postgres_data:/var/lib/postgresql/data # Persist DB data
    ports:
      - "5433:5432" # Map host port 5433 to container port 5432 (optional, for debugging)
    networks:
      - app-network # Connect to the shared network

networks:
  app-network: # Define a shared network
    driver: bridge

volumes:
  postgres_data: # Define a volume for PostgreSQL data
```

### 3.5 Practice & Testing
#### Run Application with Docker Compose
In the `demo-cicd` directory, run:
```bash
docker-compose up -d # -d: run in the background
```
This pulls the `postgres` image (if not already present), creates the network and volume, and starts both `app` and `db` containers.

#### Check Running Containers
```bash
docker-compose ps
# or
docker ps
```
You should see `spring-boot-app-compose` and `postgres-db-compose`.

#### Check Logs
```bash
docker-compose logs -f # View logs for all services
docker-compose logs -f app # View logs for the 'app' service
docker-compose logs -f db # View logs for the 'db' service
```
Check the `app` service logs for database connection errors.

#### Test
- Open a browser and visit [http://localhost:8081](http://localhost:8081). You should see "Hello from CI/CD Demo!". If the application starts without database connection errors in the logs, the setup is successful.
- (Advanced) Add an Entity and Repository to the Spring Boot code to test reading/writing data to the database via a new API endpoint.

#### Cleanup
Stop and remove containers, networks, and volumes (note: `--volumes` deletes the database volume):
```bash
docker-compose down # Stops and removes containers and networks
# or
docker-compose down --volumes # Also removes volumes
```

## Module 4: Basic Jenkins Pipeline with Jenkinsfile

### 4.1 Objectives
- Understand the basic structure of a Jenkinsfile (Declarative Pipeline).
- Write a Jenkinsfile to perform steps: Checkout code, Build application (using Maven).
- Create and run a Pipeline job in Jenkins.

### 4.2 Brief Theory
- **Jenkins Pipeline**: A suite of plugins for implementing and integrating continuous delivery pipelines in Jenkins. Pipelines can be defined as code via a Jenkinsfile.
- **Jenkinsfile**: A text file stored in the SCM (e.g., Git) alongside the project code, defining the CI/CD process.
- **Declarative Pipeline**: A newer, structured, and easier-to-read syntax compared to Scripted Pipeline.
  - `pipeline { ... }`: The outer block.
  - `agent`: Specifies the execution environment (e.g., `any` for any agent, `docker` for a Docker container).
  - `stages { ... }`: Defines the main stages of the pipeline (e.g., Build, Test, Deploy).
  - `stage('Name') { ... }`: A specific stage.
  - `steps { ... }`: Contains the actions to execute in a stage.
- Common steps:
  - `git`: Checks out code from a repository.
  - `sh`: Executes a shell command (Linux/macOS).
  - `bat`: Executes a batch command (Windows).
  - `docker.build()`: Builds a Docker image (requires Docker Pipeline plugin).
  - `echo`: Prints a message to the log.

### 4.3 Preparation
#### Create a GitHub Repository
- Create a new public or private repository on GitHub.
- Push the `demo-cicd` project code (including `src`, `pom.xml`, `Dockerfile`, `docker-compose.yml`) to the repository:
  ```bash
  # In the demo-cicd directory
  git init
  git add .
  # Create .gitignore if needed (e.g., ignore target, .idea, .vscode)
  echo "target/" >> .gitignore
  echo ".idea/" >> .gitignore
  echo ".vscode/" >> .gitignore
  git add .gitignore
  git commit -m "Initial commit"
  git branch -M main
  git remote add origin <YOUR_REPO_URL>.git
  git push -u origin main
  ```

#### Install Required Jenkins Plugins
- Access Jenkins at [http://localhost:8080](http://localhost:8080).
- Go to *Manage Jenkins -> Plugins -> Available plugins*.
- Install (if not already present):
  - Pipeline (usually pre-installed)
  - Git (usually pre-installed)
  - Docker Pipeline
  - Docker Commons
  - docker-compose (optional, for compose-specific steps)
- Restart Jenkins if prompted.

#### Configure Tools (Maven) in Jenkins (Optional but Recommended)
- Go to *Manage Jenkins -> Tools*.
- In *Maven installations*, click *Add Maven*.
- Name: e.g., `Maven 3.9.6`.
- Select *Install automatically* and choose the desired Maven version.
- Click *Save*.

### 4.4 Write Code (Jenkinsfile)
In the root `demo-cicd` directory, create a file named `Jenkinsfile` (no extension) with:
```groovy
pipeline {
    agent any // Run on any available agent (here, the Jenkins controller)

    // Define tools to use (name must match the one configured in Jenkins)
    tools {
        maven 'Maven 3.9.6' // Use the configured Maven
    }

    stages {
        stage('Checkout') { // Stage to checkout code
            steps {
                echo 'Checking out code...'
                // Automatically checkout code from the SCM configured for this job
                checkout scm
            }
        }

        stage('Build') { // Stage to build the application
            steps {
                echo 'Building the application...'
                // Execute Maven build command (using mvn from the defined tool)
                // Use sh for Linux/macOS agents; use bat for Windows
                sh "mvn clean package"
            }
        }

        // Add more stages (Test, Build Image, Deploy) in later modules
    }

    // Actions to perform after the pipeline completes (success, failure, etc.)
    post {
        always {
            echo 'Pipeline finished.'
            // Example: clean up workspace
            // cleanWs()
        }
        success {
            echo 'Pipeline successfully completed.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
```

Commit and push the Jenkinsfile to your GitHub repository:
```bash
git add Jenkinsfile
git commit -m "Add basic Jenkinsfile for build"
git push origin main
```

### 4.5 Practice & Testing
#### Create a Pipeline Job in Jenkins
- Access the Jenkins UI.
- Click *New Item* in the left menu.
- Enter a name (e.g., `demo-cicd-pipeline`).
- Select *Pipeline* and click *OK*.
- Configure the Job:
  - In *General*, add a description (optional).
  - In *Pipeline*:
    - *Definition*: Select *Pipeline script from SCM*.
    - *SCM*: Select *Git*.
    - *Repository URL*: Paste your GitHub repository’s `.git` URL.
    - *Credentials*:
      - For public repos, select *None*.
      - For private repos, click *Add -> Jenkins*, enter your GitHub username/password or SSH key, and select the credential.
    - *Branch Specifier*: Enter `*/main` (or your main branch name).
    - *Script Path*: Leave as `Jenkinsfile` (Jenkins will look for this file in the repo root).
  - Click *Save*.

#### Run the Pipeline
- On the `demo-cicd-pipeline` job page, click *Build Now* in the left menu.
- Monitor the pipeline in *Build History*. Click the build number (e.g., `#1`) for details.
- View *Console Output* for detailed logs of each step.
- The pipeline will execute the *Checkout* and *Build* stages. If successful, the *Build* stage will run `mvn clean package`.

#### Test
- Check the *Console Output* for errors.
- Ensure both *Checkout* and *Build* stages are green (successful).

## Module 5: Integrating GitHub Webhook

### 5.1 Objectives
- Understand how GitHub Webhooks work.
- Configure a GitHub Webhook to automatically trigger the Jenkins pipeline on push events.
- Verify that the automatic trigger works.

### 5.2 Brief Theory
- **Webhook**: A mechanism for one application (GitHub) to notify another (Jenkins) about an event by sending an HTTP request (typically POST) to a configured URL (Payload URL).
- **Process**:
  - A user pushes code to GitHub.
  - GitHub detects the push event.
  - GitHub sends an HTTP POST request with event details to the configured Payload URL (a specific Jenkins endpoint).
  - Jenkins receives the request, validates it (if a secret is used), analyzes the information, and triggers the corresponding pipeline job.
- **Challenge with Local Jenkins**: Jenkins running on `localhost:8080` is not accessible from the internet (where GitHub’s servers reside). A tool like `ngrok` can temporarily expose Jenkins to the internet for testing webhooks.
- **ngrok**: Creates a secure tunnel from a public URL (e.g., `https://<random-string>.ngrok.io`) to a local port (e.g., `localhost:8080`).

### 5.3 Setup (If Jenkins Runs Locally)
#### Install ngrok
- Visit [https://ngrok.com/download](https://ngrok.com/download).
- Download and extract ngrok.
- (Optional) Sign up for a free account and add your authtoken:
  ```bash
  ./ngrok config add-authtoken <YOUR_AUTHTOKEN>
  ```

#### Run ngrok
- Open a new terminal (keep it running alongside Jenkins and Docker).
- Create a tunnel to the Jenkins port (8080):
  ```bash
  ./ngrok http 8080
  ```
- ngrok will display a public URL, e.g., `Forwarding https://<random-string>.ngrok.io -> http://localhost:8080`. Copy the `https://...ngrok.io` URL. This is your temporary public Jenkins URL.

### 5.4 Configure GitHub Webhook
- Access your GitHub repository’s *Settings*:
  - Open your repository on GitHub.
  - Click the *Settings* tab.
  - Select *Webhooks* in the left menu.
  - Click *Add webhook*.
- Fill in Webhook details:
  - *Payload URL*:
    - If using ngrok: Paste the ngrok URL and append `/github-webhook/`, e.g., `https://<random-string>.ngrok.io/github-webhook/`.
    - If Jenkins has a public IP/domain: Use that + `/github-webhook/`, e.g., `http://your-jenkins.com/github-webhook/`.
  - *Content type*: Select `application/json`.
  - *Secret*: (Highly recommended for security) Enter a strong, random string (e.g., generated by a password manager). Copy this secret.
  - *Which events would you like to trigger this webhook?*: Select *Just the push event* (default).
  - *Active*: Ensure this is checked.
  - Click *Add webhook*.

### 5.5 Configure Jenkins Job
- Access the `demo-cicd-pipeline` job in Jenkins.
- Click *Configure* in the left menu.
- Enable Trigger:
  - In *Build Triggers*, check *GitHub hook trigger for GITScm polling*.
  - (No secret configuration is needed if using the GitHub plugin).
- Click *Save*.

### 5.6 Testing
#### Check Webhook Delivery on GitHub
- Go to *Settings -> Webhooks* in your GitHub repository.
- You’ll see the webhook you created. GitHub automatically sends a “ping” event to the Payload URL. Click the webhook and scroll to *Recent Deliveries*. A green checkmark indicates successful delivery to Jenkins (via ngrok). A red mark means to check the URL and ensure ngrok is running.

#### Trigger Pipeline by Pushing Code
- Make a small change to your code (e.g., edit `README.md` or add a comment).
- Commit and push to the `main` branch:
  ```bash
  # Example: Edit README.md
  echo "Update for webhook test" >> README.md
  git add README.md
  git commit -m "Test GitHub webhook trigger"
  git push origin main
  ```

#### Observe Jenkins
- After pushing, return to the `demo-cicd-pipeline` job in Jenkins.
- A new build should appear in *Build History* within seconds.
- Check the *Console Output* of the new build to ensure it ran successfully.
- On GitHub, check *Webhook deliveries* for a new push event with a green checkmark.

## Module 6: Completing the CI/CD Pipeline: Build Image & Deploy

### 6.1 Objectives
- Extend the Jenkinsfile to add stages: Build Docker Image, Push Docker Image (optional), Deploy with Docker Compose.
- Understand how to use Docker and Docker Compose commands within a Jenkinsfile.
- Automatically deploy the application to an environment (the Jenkins/Docker machine in this demo).

### 6.2 Brief Theory
- **Build Docker Image in Pipeline**: Jenkins needs access to the Docker daemon. The way we run the Jenkins container (`-v /var/run/docker.sock:/var/run/docker.sock`) enables this. Use `sh 'docker build ...'` or the Docker Pipeline plugin (`docker.build()`).
- **Tagging Image**: Tag images consistently, e.g., using the build number (`${env.BUILD_NUMBER}`) or commit hash (`${env.GIT_COMMIT}`) for version management.
- **Push Image (Optional)**: In real-world scenarios, images are pushed to a Docker registry (Docker Hub, AWS ECR, Google GCR, etc.) for use in other environments (staging, production). Registry credentials must be configured in Jenkins.
- **Deploy with Docker Compose**: Use `sh 'docker-compose down && docker-compose up -d'` to stop the old version and start the new one with the newly built image. Ensure `docker-compose.yml` uses a dynamic image tag or is updated before running `up`.

### 6.3 Setup
#### Configure Docker Hub Credentials (If Pushing Images)
- Go to *Manage Jenkins -> Credentials*.
- Click *(global)* (or the appropriate domain).
- Click *Add Credentials*.
  - *Kind*: Select *Username with password*.
  - *Scope*: Global.
  - *Username*: Your Docker Hub username.
  - *Password*: Your Docker Hub password or Access Token (recommended).
  - *ID*: Set a memorable ID (e.g., `dockerhub-creds`).
  - *Description*: E.g., “Docker Hub Credentials”.
  - Click *Create*.

### 6.4 Update Code
#### Update `docker-compose.yml` (For Dynamic Image Tags)
Modify `docker-compose.yml` to use an environment variable for the `app` service image instead of hardcoding `image: demo-cicd-app:latest`:
```yaml
version: '3.8'

services:
  app:
    # Use APP_IMAGE environment variable, fallback to default if not set
    image: ${APP_IMAGE:-demo-cicd-app:latest}
    container_name: spring-boot-app-compose
    ports:
      - "8081:8080"
    environment:
      SPRING_PROFILES_ACTIVE: default
    depends_on:
      - db
    networks:
      - app-network

  db:
    image: postgres:15
    container_name: postgres-db-compose
    environment:
      POSTGRES_DB: mydatabase
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5433:5432"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
```

Commit and push the updated `docker-compose.yml` to GitHub.

#### Update Jenkinsfile
Add new stages for *Build Docker Image* and *Deploy*. Include *Push Docker Image* if desired:
```groovy
pipeline {
    agent any

    tools {
        maven 'Maven 3.9.6'
    }

    // Environment variables used in the pipeline
    environment {
        // Docker image name (replace 'your-dockerhub-username' if pushing)
        // If not pushing, keep it simple like 'demo-cicd-app'
        // DOCKER_IMAGE_NAME = 'your-dockerhub-username/demo-cicd-app'
        DOCKER_IMAGE_NAME = 'demo-cicd-app'
        // Credential ID created in Jenkins for Docker Hub (only needed if pushing)
        DOCKERHUB_CREDENTIAL_ID = 'dockerhub-creds'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }

        stage('Build App') { // Renamed for clarity
            steps {
                echo 'Building the application...'
                sh "mvn clean package"
            }
        }

        stage('Build Docker Image') {
            steps {
                script { // Use script block for variable handling
                    // Create tag using Build Number
                    def imageTag = "${env.BUILD_NUMBER}"
                    // Full image name with tag
                    def fullImageName = "${env.DOCKER_IMAGE_NAME}:${imageTag}"
                    echo "Building Docker image: ${fullImageName}"
                    // Build image using docker command
                    sh "docker build -t ${fullImageName} ."

                    // Store image name for Deploy stage
                    env.APP_IMAGE_TO_DEPLOY = fullImageName
                }
            }
        }

        /*
        // --- PUSH IMAGE STAGE (OPTIONAL) ---
        stage('Push Docker Image') {
            // Only run on main branch (example)
            // when { branch 'main' }
            steps {
                script {
                    // Ensure APP_IMAGE_TO_DEPLOY is set
                    if (env.APP_IMAGE_TO_DEPLOY) {
                        echo "Pushing image ${env.APP_IMAGE_TO_DEPLOY} to Docker Hub..."
                        // Use Docker Pipeline plugin to push with credentials
                        docker.withRegistry('https://registry.hub.docker.com', env.DOCKERHUB_CREDENTIAL_ID) {
                            docker.image(env.APP_IMAGE_TO_DEPLOY).push()
                        }
                        // Optionally push 'latest' tag (typically for main branch)
                        // docker.image(env.APP_IMAGE_TO_DEPLOY).push('latest')
                    } else {
                        error "APP_IMAGE_TO_DEPLOY variable not set!"
                    }
                }
            }
        }
        */

        stage('Deploy') {
            steps {
                script {
                    if (env.APP_IMAGE_TO_DEPLOY) {
                        echo "Deploying image ${env.APP_IMAGE_TO_DEPLOY} using Docker Compose..."
                        // Run docker-compose with the image name passed via APP_IMAGE
                        sh """
                           export APP_IMAGE=${env.APP_IMAGE_TO_DEPLOY}
                           docker-compose down
                           docker-compose up -d
                        """
                    } else {
                        error "APP_IMAGE_TO_DEPLOY variable not set for deployment!"
                    }
                }
            }
        }
    } // end stages

    post {
        always {
            echo 'Pipeline finished.'
            // Optionally clean up old images to save space
            // sh "docker image prune -f"
        }
        success {
            echo "Pipeline successfully completed for image: ${env.APP_IMAGE_TO_DEPLOY}"
        }
        failure {
            echo 'Pipeline failed.'
        }
    } // end post
}
```

**Explanation**:
- `environment { ... }`: Defines shared variables. `DOCKER_IMAGE_NAME` is the image name (add username if pushing to Docker Hub). `DOCKERHUB_CREDENTIAL_ID` is the ID of the created credential.
- `stage('Build Docker Image')`: Uses `sh "docker build ..."` to build the image. Tags it with `${env.BUILD_NUMBER}`. Stores the full image name in `env.APP_IMAGE_TO_DEPLOY` for later use.
- `stage('Push Docker Image')` (Commented out, enable if needed): Uses `docker.withRegistry` to log in to the registry and `docker.image(...).push()` to push.
- `stage('Deploy')`: Runs `docker-compose` with `export APP_IMAGE=${env.APP_IMAGE_TO_DEPLOY}` to pass the image name to `docker-compose.yml`. Runs `docker-compose down` to stop old containers, then `docker-compose up -d` to start with the new image.

Commit and push the updated Jenkinsfile to GitHub.

### 6.5 Practice & Testing
#### Trigger Pipeline
- Pushing the updated Jenkinsfile (and `docker-compose.yml`) to GitHub will trigger the pipeline (via the webhook). Alternatively, click *Build Now* manually.
- Monitor the pipeline:
  - Watch the new stages: *Build App*, *Build Docker Image*, *Deploy* (and *Push Docker Image* if enabled).
  - Check *Console Output* for errors.
  - *Build Docker Image* will show logs of the image build.
  - *Deploy* will show logs of `docker-compose down` and `docker-compose up -d`.

#### Test Application
- After the pipeline completes successfully, visit [http://localhost:8081](http://localhost:8081). The application should be running with the latest code.
- Run `docker ps` to verify that `spring-boot-app-compose` and `postgres-db-compose` are running. Check the *IMAGE* column for `spring-boot-app-compose` to confirm it’s using the image with the latest build number (e.g., `demo-cicd-app:5`).
- Repeat Test: Modify `HelloController.java` (e.g., change the returned string), commit, and push to GitHub. Watch the pipeline automatically run and deploy the new version. Visit [http://localhost:8081](http://localhost:8081) to see the change.

## Module 7: Summary and Next Steps

### 7.1 Summary
Congratulations! You’ve completed the course and built a fully functional basic CI/CD pipeline:
- Automatically triggered by code pushes to GitHub.
- Checks out the latest code.
- Builds a Spring Boot application with Maven.
- Builds a Docker image for the application.
- (Optional) Pushes the image to Docker Hub.
- Deploys the application (and database) using Docker Compose.
You’ve mastered the core tools: Jenkins (Pipeline/Jenkinsfile), Docker, Docker Compose, and GitHub Webhooks.

### 7.2 Next Steps
This pipeline is a great starting point, but there are many ways to improve and extend it:
- **Add Test Stage**: Include a *Test* stage after *Build App* to run unit tests (`mvn test`) or integration tests. Only proceed if tests pass.
- **Use Docker Agent**: Instead of `agent any`, use `agent { docker { image 'maven:3.9-eclipse-temurin-17' } }` to run the *Build* stage in a separate Maven container, keeping the Jenkins controller cleaner. Similarly for other stages.
- **Manage Secrets**: Avoid hardcoding database passwords in `docker-compose.yml` or `application.properties`. Use Jenkins Credentials to manage secrets and pass them to containers as environment variables or files.
- **Multiple Environments (Staging, Production)**: Create separate pipelines or jobs for different environments. Use different Git branches (e.g., `develop`, `main`) to trigger corresponding pipelines.
- **Deploy to Remote Servers**: Instead of deploying on the Jenkins machine, use plugins like SSH Agent or Publish Over SSH to build the image on Jenkins, push to a registry, then SSH to the target server to pull and run `docker-compose up`.
- **Notifications**: Configure Jenkins to send notifications (Email, Slack, Teams) on pipeline success or failure.
- **Security Scanning**: Integrate tools like SonarQube for code scanning or Trivy/Snyk for Docker image vulnerability scanning.
- **Advanced Deployment Strategies**: Implement Blue/Green Deployment or Canary Release to minimize downtime and risks.
- **Infrastructure as Code (IaC)**: Use tools like Terraform or Ansible to automate infrastructure management (servers, networks, databases).

We hope this course has been valuable and provides a strong foundation for exploring the world of CI/CD and DevOps!