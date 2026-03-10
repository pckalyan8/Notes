# Phase 15.3 — CI/CD Pipelines for Java

## The Philosophy Behind CI/CD

**Continuous Integration (CI)** is the practice of merging every developer's code into the shared repository frequently — ideally multiple times per day — and automatically validating each merge with a build and test suite. The goal is to detect integration problems early, when they are cheap to fix, rather than discovering them during a big-bang release.

**Continuous Delivery (CD)** extends CI by automatically deploying every validated build to a staging environment. **Continuous Deployment** goes one step further and deploys every validated build straight to production without human approval.

The mental model is a pipeline: code flows in one end (a `git push`), passes through a series of automated quality gates, and deployable artifacts emerge from the other end.

---

## A Typical Java CI/CD Pipeline

A complete pipeline for a Spring Boot project looks like this:

```
Developer pushes code
        │
        ▼
[1] Checkout & Compile     → Catch syntax errors immediately
        │
        ▼
[2] Unit Tests             → Fast feedback, mocked dependencies
        │
        ▼
[3] Static Analysis        → SonarQube, Checkstyle, SpotBugs
        │
        ▼
[4] Integration Tests      → Real DB via Testcontainers
        │
        ▼
[5] Build Docker Image     → Multi-stage, tagged with git SHA
        │
        ▼
[6] Push to Registry       → Docker Hub, ECR, GCR
        │
        ▼
[7] Deploy to Staging      → Helm upgrade / kubectl apply
        │
        ▼
[8] Smoke / E2E Tests      → Verify deployed app responds correctly
        │
        ▼
[9] Deploy to Production   → Manual approval gate (Continuous Delivery)
                             OR automatic (Continuous Deployment)
```

---

## GitHub Actions — The Modern Standard

GitHub Actions is the most widely adopted CI/CD system for open-source and cloud-native Java projects. Pipelines are defined in YAML files inside `.github/workflows/`. Each workflow is triggered by events (push, pull request, schedule, etc.) and runs on **runners** (GitHub-hosted virtual machines or self-hosted servers).

### Complete Java CI Workflow

```yaml
# .github/workflows/ci.yml
name: Java CI Pipeline

# Trigger on pushes to main or any pull request
on:
  push:
    branches: [ "main", "develop" ]
  pull_request:
    branches: [ "main" ]

jobs:
  # ────────────────────────────────────────────────────
  # JOB 1: Build and Test
  # ────────────────────────────────────────────────────
  build-and-test:
    runs-on: ubuntu-latest   # GitHub-hosted runner (Ubuntu)

    services:
      # Spin up a real PostgreSQL container alongside the job
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      # Step 1: Check out the repository code
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Set up a specific JDK version
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'       # Eclipse Temurin (AdoptOpenJDK successor)
          cache: maven                  # Cache ~/.m2/repository between runs

      # Step 3: Run unit tests
      - name: Run unit tests
        run: mvn test -pl . -am

      # Step 4: Run integration tests with the real PostgreSQL service above
      - name: Run integration tests
        run: mvn verify -P integration-test
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_PASSWORD: testpassword

      # Step 5: Generate and upload coverage report
      - name: Generate JaCoCo coverage report
        run: mvn jacoco:report

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: target/site/jacoco/jacoco.xml

      # Step 6: Upload build artifact (the JAR)
      - name: Upload JAR artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar

  # ────────────────────────────────────────────────────
  # JOB 2: Static Code Analysis (runs in parallel with build-and-test)
  # ────────────────────────────────────────────────────
  code-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # Required by SonarQube for blame information

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Analyze with SonarCloud
        run: mvn sonar:sonar
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}  # Stored in GitHub Secrets
          SONAR_HOST_URL: https://sonarcloud.io

  # ────────────────────────────────────────────────────
  # JOB 3: Build and Push Docker Image (only on main branch)
  # ────────────────────────────────────────────────────
  docker-build-push:
    runs-on: ubuntu-latest
    needs: [build-and-test, code-analysis]   # Only runs if both jobs above succeed
    if: github.ref == 'refs/heads/main'      # Only on the main branch

    steps:
      - uses: actions/checkout@v4

      # Login to DockerHub (credentials stored in GitHub Secrets)
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # Extract metadata for image tagging
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: myorg/myapp
          tags: |
            type=sha,prefix=sha-         # Tag with git commit SHA (e.g., sha-a1b2c3d)
            type=ref,event=branch        # Tag with branch name
            type=semver,pattern={{version}}  # Tag with semantic version (from git tag)

      # Build and push the image using Docker Buildx (enables caching)
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          # Cache layers in GitHub Actions cache to speed up builds
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ────────────────────────────────────────────────────
  # JOB 4: Deploy to Staging
  # ────────────────────────────────────────────────────
  deploy-staging:
    runs-on: ubuntu-latest
    needs: docker-build-push
    environment: staging   # GitHub Environment with optional protection rules

    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG_STAGING }}

      - name: Deploy to staging via Helm
        run: |
          helm upgrade --install myapp ./helm/myapp \
            --namespace staging \
            --set image.tag=sha-${{ github.sha }} \
            --set replicaCount=2 \
            --wait               # Wait for rollout to complete before declaring success
            --timeout=5m
```

---

## Jenkins — Enterprise CI/CD

Jenkins is the veteran of Java CI/CD, widely used in enterprise environments. Pipelines are defined in a **Jenkinsfile** using Groovy DSL and typically checked into the repository root.

```groovy
// Jenkinsfile
pipeline {
    agent {
        // Run inside a Docker container with Maven and JDK pre-installed
        docker {
            image 'maven:3.9-eclipse-temurin-21'
            args '-v /root/.m2:/root/.m2'  // Mount Maven cache from host
        }
    }

    environment {
        // Credentials stored in Jenkins Credential Store
        DOCKER_CREDENTIALS = credentials('dockerhub-credentials')
        SONAR_TOKEN        = credentials('sonar-token')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile -q'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    // Publish JUnit test results to Jenkins UI
                    junit 'target/surefire-reports/*.xml'
                    // Publish JaCoCo coverage report
                    jacoco execPattern: 'target/jacoco.exec'
                }
            }
        }

        stage('Code Quality') {
            steps {
                sh """
                    mvn sonar:sonar \
                      -Dsonar.host.url=https://sonarcloud.io \
                      -Dsonar.token=${SONAR_TOKEN}
                """
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Docker Build & Push') {
            when {
                branch 'main'   // Only run on the main branch
            }
            steps {
                script {
                    def imageTag = "myorg/myapp:${env.GIT_COMMIT[0..7]}"
                    sh "docker build -t ${imageTag} ."
                    sh "echo ${DOCKER_CREDENTIALS_PSW} | docker login -u ${DOCKER_CREDENTIALS_USR} --password-stdin"
                    sh "docker push ${imageTag}"
                }
            }
        }

        stage('Deploy to Staging') {
            when { branch 'main' }
            steps {
                sh """
                    helm upgrade --install myapp ./helm/myapp \
                      --namespace staging \
                      --set image.tag=${env.GIT_COMMIT[0..7]} \
                      --wait
                """
            }
        }

        stage('Deploy to Production') {
            when { branch 'main' }
            // Require manual approval before deploying to production
            input {
                message "Deploy to production?"
                ok "Yes, deploy!"
                submitter "team-leads"   // Only team leads can approve
            }
            steps {
                sh """
                    helm upgrade --install myapp ./helm/myapp \
                      --namespace production \
                      --set image.tag=${env.GIT_COMMIT[0..7]} \
                      --set replicaCount=5 \
                      --wait
                """
            }
        }
    }

    post {
        failure {
            // Notify Slack on failure
            slackSend channel: '#deployments',
                      color: 'danger',
                      message: "Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        success {
            slackSend channel: '#deployments',
                      color: 'good',
                      message: "Pipeline SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
```

---

## GitLab CI — `.gitlab-ci.yml`

GitLab CI is tightly integrated with GitLab repositories and uses a concept called **stages** where all jobs in a stage must succeed before the next stage begins.

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - analyze
  - package
  - deploy

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  DOCKER_IMAGE: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"

# Cache Maven dependencies across jobs using GitLab's cache mechanism
cache:
  paths:
    - .m2/repository/

# Compile the project
compile:
  stage: build
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn compile -q

# Run unit tests and generate coverage
unit-test:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn test jacoco:report
  artifacts:
    when: always
    reports:
      junit: target/surefire-reports/*.xml    # GitLab shows test results in MR
    paths:
      - target/site/jacoco/

# SonarQube analysis
sonar:
  stage: analyze
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn sonar:sonar -Dsonar.token=$SONAR_TOKEN
  only:
    - main
    - merge_requests

# Build Docker image and push to GitLab Container Registry
docker-build:
  stage: package
  image: docker:24
  services:
    - docker:24-dind            # Docker-in-Docker service
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
    - docker build -t "$DOCKER_IMAGE" .
    - docker push "$DOCKER_IMAGE"
  only:
    - main

# Deploy to staging
deploy-staging:
  stage: deploy
  image: alpine/helm:3.12.0
  script:
    - helm upgrade --install myapp ./helm/myapp
        --set image.tag=$CI_COMMIT_SHORT_SHA
        --namespace staging
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main

# Deploy to production with manual trigger
deploy-production:
  stage: deploy
  image: alpine/helm:3.12.0
  script:
    - helm upgrade --install myapp ./helm/myapp
        --set image.tag=$CI_COMMIT_SHORT_SHA
        --namespace production
  environment:
    name: production
    url: https://www.example.com
  when: manual              # Requires a human to click "Deploy" in GitLab UI
  only:
    - main
```

---

## Blue-Green and Canary Deployments

### Blue-Green Deployment

In a blue-green deployment you maintain **two identical production environments** — blue (currently live) and green (new version). When you deploy a new version to green and validate it, you flip the load balancer to point at green. If something goes wrong, the rollback is instantaneous: just flip the load balancer back to blue.

```yaml
# Blue Service (currently active)
apiVersion: v1
kind: Service
metadata:
  name: myapp-production
spec:
  selector:
    app: myapp
    slot: blue      # Change to "green" to flip traffic
  ports:
    - port: 80
      targetPort: 8080
```

### Canary Deployment

A **canary** sends a small percentage of traffic (e.g., 5%) to the new version while the majority goes to the stable version. If the canary shows no errors, the percentage is gradually increased to 100%. This is the safest deployment strategy for high-traffic production systems and is natively supported by service meshes like Istio and by ingress controllers like NGINX.

---

## Best Practices Summary

**Fail fast.** Put the fastest jobs (compilation, unit tests) first so developers get feedback in under 3 minutes. Slow integration tests should run later in the pipeline.

**Cache aggressively.** Maven downloads hundreds of megabytes of dependencies. Always cache `~/.m2/repository` across pipeline runs. In GitHub Actions, the `setup-java` action handles this automatically when you set `cache: maven`.

**Never bake secrets into images.** Use environment variables, Kubernetes Secrets, or a vault. Treat your Docker image as if it will be public.

**Tag images with the git commit SHA**, not just `latest`. This makes every deployed version traceable and makes rollbacks trivial.

**Separate CI (build/test) from CD (deploy).** Use branch protections so that only validated, reviewed code can be deployed to production. Require the CI pipeline to pass before merging any pull request.

**Test at every stage**, not just in unit tests. Run smoke tests against your staging deployment to catch issues that only appear in a real container/network environment.

**Make pipelines idempotent.** Running the same pipeline twice should produce the same result. Avoid time-based tags or mutable state in your pipeline logic.
