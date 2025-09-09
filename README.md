# Jenkins Cheat Sheet

## Table of Contents
- [Installation and Setup](#installation-and-setup)
- [Jobs and Builds](#jobs-and-builds)
- [Pipelines](#pipelines)
- [Plugins](#plugins)
- [Groovy Scripting](#groovy-scripting)
- [Integration with Other Tools](#integration-with-other-tools)
- [Best Practices and Security](#best-practices-and-security)
- [Troubleshooting Tips](#troubleshooting-tips)

## Installation and Setup

### Installation Methods

#### Linux (Ubuntu/Debian)
```bash
# Add Jenkins repository
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io.key | sudo tee \
    /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update package index
sudo apt-get update

# Install Java (required)
sudo apt install openjdk-11-jdk

# Install Jenkins
sudo apt-get install jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Check status
sudo systemctl status jenkins
```

#### Docker Installation
```bash
# Run Jenkins in Docker
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts

# With Docker compose
cat > docker-compose.yml << EOF
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
    environment:
      - JAVA_OPTS=-Dhudson.footerURL=http://mycompany.com
volumes:
  jenkins_home:
EOF

docker-compose up -d
```

#### Configuration
```bash
# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Jenkins directories
/var/lib/jenkins/          # Jenkins home
/var/lib/jenkins/jobs/     # Job configurations
/var/lib/jenkins/plugins/  # Installed plugins
/var/lib/jenkins/workspace/ # Job workspaces
/var/log/jenkins/          # Log files

# Configuration files
/etc/default/jenkins       # Service configuration
/etc/init.d/jenkins        # Service script
```

### Basic Configuration
```bash
# Set Jenkins URL
# Manage Jenkins > Configure System > Jenkins Location

# Configure security
# Manage Jenkins > Configure Global Security

# Install plugins
# Manage Jenkins > Manage Plugins

# Configure tools
# Manage Jenkins > Global Tool Configuration
```

## Jobs and Builds

### Freestyle Jobs
```bash
# Create new freestyle job
# New Item > Freestyle project

# Source Code Management (Git example)
Repository URL: https://github.com/user/repo.git
Credentials: [select or add credentials]
Branch Specifier: */main

# Build Triggers
# - Build after other projects are built
# - Build periodically: H/15 * * * * (every 15 minutes)
# - GitHub hook trigger for GITScm polling
# - Poll SCM: H/5 * * * * (poll every 5 minutes)

# Build Environment
# - Delete workspace before build starts
# - Use secret text(s) or file(s)
# - Add timestamps to the Console Output

# Build Steps
# Execute shell:
#!/bin/bash
set -e
echo "Starting build..."
npm install
npm test
npm run build

# Post-build Actions
# - Archive the artifacts: dist/**
# - Publish JUnit test result report: reports/junit.xml
# - Send build artifacts over SSH
# - Email notification
```

### Parameterized Jobs
```groovy
// String parameter
String BRANCH_NAME
Default: main
Description: Branch to build

// Choice parameter  
Choice ENVIRONMENT
Choices: dev\nstaging\nproduction
Description: Target environment

// Boolean parameter
Boolean SKIP_TESTS
Default: false
Description: Skip running tests

// File parameter
File DEPLOYMENT_CONFIG
Description: Upload deployment configuration

// Using parameters in build
#!/bin/bash
echo "Building branch: ${BRANCH_NAME}"
echo "Target environment: ${ENVIRONMENT}"
if [ "${SKIP_TESTS}" = "true" ]; then
    echo "Skipping tests"
else
    npm test
fi
```

### Multi-configuration Jobs
```bash
# Matrix project configuration
# Configuration Matrix:

# Axis 1 - User-defined Axis
Name: OS
Values: linux windows macos

# Axis 2 - JDK Axis  
Name: JDK
Values: openjdk8 openjdk11 openjdk17

# Axis 3 - User-defined Axis
Name: BROWSER
Values: chrome firefox safari

# Combination Filter
(OS=="linux" && BROWSER!="safari") || (OS=="windows" && BROWSER!="safari") || (OS=="macos")

# Build script using matrix variables
#!/bin/bash
echo "Running on OS: ${OS}"
echo "Using JDK: ${JDK}"
echo "Testing with browser: ${BROWSER}"
```

## Pipelines

### Declarative Pipeline
```groovy
pipeline {
    agent any
    
    environment {
        APP_NAME = 'myapp'
        VERSION = '1.0.0'
        REGISTRY = 'docker.io/myuser'
    }
    
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'production'],
            description: 'Target environment'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip running tests'
        )
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/user/repo.git'
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    echo "Building ${APP_NAME} version ${VERSION}"
                    npm install
                    npm run build
                '''
            }
        }
        
        stage('Test') {
            when {
                not { params.SKIP_TESTS }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm run test:unit'
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'reports/junit.xml'
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    def scanResult = sh(
                        script: 'npm audit --audit-level high',
                        returnStatus: true
                    )
                    if (scanResult != 0) {
                        unstable('Security vulnerabilities found')
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    def image = docker.build("${REGISTRY}/${APP_NAME}:${VERSION}")
                    docker.withRegistry('https://registry-1.docker.io/v2/', 'dockerhub-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy') {
            when {
                anyOf {
                    branch 'main'
                    expression { params.ENVIRONMENT == 'production' }
                }
            }
            steps {
                script {
                    if (params.ENVIRONMENT == 'production') {
                        timeout(time: 10, unit: 'MINUTES') {
                            input message: 'Deploy to production?', ok: 'Deploy'
                        }
                    }
                }
                
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${REGISTRY}/${APP_NAME}:${VERSION} \
                        --namespace=${params.ENVIRONMENT}
                """
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            mail to: 'team@company.com',
                 subject: "Build Successful: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                 body: "Build completed successfully."
        }
        failure {
            mail to: 'team@company.com',
                 subject: "Build Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                 body: "Build failed. Check console output."
        }
        unstable {
            slackSend channel: '#dev-alerts',
                     color: 'warning',
                     message: "Build unstable: ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
        }
    }
}
```

### Scripted Pipeline
```groovy
node {
    def app
    def buildVersion = "${BUILD_NUMBER}"
    
    try {
        stage('Checkout') {
            checkout scm
        }
        
        stage('Build') {
            sh 'npm install'
            sh 'npm run build'
        }
        
        stage('Test') {
            parallel(
                "Unit Tests": {
                    sh 'npm run test:unit'
                    publishTestResults testResultsPattern: 'reports/junit.xml'
                },
                "Lint": {
                    sh 'npm run lint'
                },
                "Security Scan": {
                    sh 'npm audit'
                }
            )
        }
        
        stage('Package') {
            app = docker.build("myapp:${buildVersion}")
        }
        
        stage('Deploy to Staging') {
            app.withRun("--name staging-${buildVersion} -p 3001:3000") { c ->
                sh 'sleep 10'  // Wait for app to start
                sh 'curl -f http://localhost:3001/health || exit 1'
            }
            
            // Deploy to staging environment
            sh """
                docker stop staging || true
                docker rm staging || true
                docker run -d --name staging -p 3001:3000 myapp:${buildVersion}
            """
        }
        
        stage('Integration Tests') {
            sh 'npm run test:e2e'
        }
        
        stage('Deploy to Production') {
            if (env.BRANCH_NAME == 'main') {
                timeout(time: 5, unit: 'MINUTES') {
                    input message: 'Deploy to production?', ok: 'Deploy',
                          submitterParameter: 'DEPLOYER'
                }
                
                echo "Deploying to production by ${env.DEPLOYER}"
                
                docker.withRegistry('https://registry.company.com', 'docker-registry') {
                    app.push("${buildVersion}")
                    app.push("latest")
                }
                
                // Rolling deployment
                sh """
                    kubectl set image deployment/myapp \
                        myapp=registry.company.com/myapp:${buildVersion} \
                        --namespace=production
                    kubectl rollout status deployment/myapp --namespace=production
                """
            }
        }
        
        currentBuild.result = 'SUCCESS'
        
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        throw e
        
    } finally {
        // Cleanup
        sh 'docker system prune -f'
        
        // Notifications
        if (currentBuild.result == 'SUCCESS') {
            slackSend channel: '#deployments',
                     color: 'good',
                     message: "✅ Build successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        } else {
            slackSend channel: '#dev-alerts',
                     color: 'danger',
                     message: "❌ Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
```

### Pipeline Libraries
```groovy
// vars/deployApp.groovy (shared library)
def call(Map config) {
    pipeline {
        agent any
        
        stages {
            stage('Build') {
                steps {
                    script {
                        buildApp(config.appName, config.version)
                    }
                }
            }
            
            stage('Deploy') {
                steps {
                    script {
                        deployToEnvironment(config.environment, config.appName, config.version)
                    }
                }
            }
        }
    }
}

def buildApp(appName, version) {
    sh """
        docker build -t ${appName}:${version} .
        docker tag ${appName}:${version} ${appName}:latest
    """
}

def deployToEnvironment(env, appName, version) {
    sh """
        kubectl set image deployment/${appName} \
            ${appName}=${appName}:${version} \
            --namespace=${env}
    """
}

// Using the shared library
@Library('my-pipeline-library') _

deployApp([
    appName: 'myapp',
    version: env.BUILD_NUMBER,
    environment: 'staging'
])
```

## Plugins

### Essential Plugins
```bash
# Core plugins to install:

# Build and Deploy
- Pipeline
- Pipeline: Stage View
- Blue Ocean
- Docker Pipeline
- Kubernetes CLI
- Deploy to container

# Source Control
- Git
- GitHub
- GitLab
- Bitbucket

# Build Tools
- Gradle
- Maven Integration
- NodeJS
- Ant

# Testing and Quality
- JUnit
- Cobertura
- SonarQube Scanner
- Checkstyle
- SpotBugs

# Notifications
- Email Extension
- Slack Notification
- Microsoft Teams Notification

# Security
- Credentials Binding
- OWASP Markup Formatter
- Build Timeout

# Utilities
- Workspace Cleanup
- Timestamper
- Build Name and Description Setter
- Environment Injector
```

### Plugin Configuration Examples
```groovy
// Slack plugin configuration
slackSend(
    channel: '#dev-team',
    color: 'good',
    message: """
        Build Successful! :white_check_mark:
        Job: ${env.JOB_NAME}
        Build: ${env.BUILD_NUMBER}
        Duration: ${currentBuild.durationString}
    """,
    teamDomain: 'mycompany',
    token: env.SLACK_TOKEN
)

// Email plugin
emailext (
    subject: "Build ${currentBuild.result}: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
    body: '''
        <h2>Build Summary</h2>
        <p>Job: ${JOB_NAME}</p>
        <p>Build Number: ${BUILD_NUMBER}</p>
        <p>Build Status: ${BUILD_STATUS}</p>
        <p>Build URL: <a href="${BUILD_URL}">${BUILD_URL}</a></p>
        
        <h3>Changes</h3>
        ${CHANGES_SINCE_LAST_SUCCESS}
        
        <h3>Console Output</h3>
        <pre>${BUILD_LOG}</pre>
    ''',
    to: '${DEFAULT_RECIPIENTS}',
    recipientProviders: [developers(), requestor()]
)

// SonarQube integration
withSonarQubeEnv('SonarQube') {
    sh '''
        sonar-scanner \
            -Dsonar.projectKey=myapp \
            -Dsonar.sources=src \
            -Dsonar.host.url=${SONAR_HOST_URL} \
            -Dsonar.login=${SONAR_AUTH_TOKEN}
    '''
}

// Quality gate check
timeout(time: 10, unit: 'MINUTES') {
    def qg = waitForQualityGate()
    if (qg.status != 'OK') {
        error "Pipeline aborted due to quality gate failure: ${qg.status}"
    }
}
```

## Groovy Scripting

### Basic Groovy Syntax
```groovy
// Variables and data types
def string = "Hello, World!"
def number = 42
def boolean = true
def list = [1, 2, 3, 4, 5]
def map = [name: "John", age: 30, city: "New York"]

// Functions
def greet(name) {
    return "Hello, ${name}!"
}

// Classes
class Person {
    String name
    int age
    
    def greet() {
        return "Hi, I'm ${name} and I'm ${age} years old"
    }
}

def person = new Person(name: "Alice", age: 25)
println person.greet()

// Closures
def numbers = [1, 2, 3, 4, 5]
def doubled = numbers.collect { it * 2 }
def filtered = numbers.findAll { it > 3 }

// Exception handling
try {
    def result = 10 / 0
} catch (ArithmeticException e) {
    println "Division by zero: ${e.message}"
} finally {
    println "Cleanup code here"
}
```

### Jenkins-specific Groovy
```groovy
// Environment variables
println "Build number: ${env.BUILD_NUMBER}"
println "Job name: ${env.JOB_NAME}"
println "Workspace: ${env.WORKSPACE}"

// Build parameters
if (params.DEPLOY_TO_PROD) {
    println "Deploying to production"
}

// Current build information
println "Build result: ${currentBuild.result}"
println "Build duration: ${currentBuild.durationString}"
println "Build URL: ${env.BUILD_URL}"

// Accessing build causes
def causes = currentBuild.getBuildCauses()
for (cause in causes) {
    println "Build cause: ${cause}"
}

// Working with files
def workspace = env.WORKSPACE
def file = new File("${workspace}/config.properties")
if (file.exists()) {
    def properties = new Properties()
    file.withInputStream { stream ->
        properties.load(stream)
    }
    println "Version: ${properties.version}"
}

// HTTP requests
@Grab('org.apache.httpcomponents:httpclient:4.5.13')
import org.apache.http.impl.client.HttpClients
import org.apache.http.client.methods.HttpGet

def client = HttpClients.createDefault()
def request = new HttpGet("https://api.github.com/repos/user/repo")
def response = client.execute(request)
println "Status: ${response.getStatusLine().getStatusCode()}"

// JSON processing
@Grab('org.apache.groovy:groovy-json:4.0.0')
import groovy.json.JsonSlurper
import groovy.json.JsonBuilder

def jsonSlurper = new JsonSlurper()
def data = jsonSlurper.parseText('{"name": "John", "age": 30}')
println "Name: ${data.name}"

def builder = new JsonBuilder()
builder {
    name "Alice"
    age 25
    skills(["Java", "Groovy", "Jenkins"])
}
println builder.toString()
```

### Script Console Examples
```groovy
// Get all jobs
Jenkins.instance.getAllItems(Job.class).each { job ->
    println "Job: ${job.name}, Last build: ${job.lastBuild?.number ?: 'None'}"
}

// Find jobs by pattern
def pattern = ~/.*api.*/
Jenkins.instance.getAllItems(Job.class).findAll { job ->
    job.name ==~ pattern
}.each { job ->
    println "Matching job: ${job.name}"
}

// Disable all jobs containing 'test' in name
Jenkins.instance.getAllItems(Job.class).findAll { job ->
    job.name.toLowerCase().contains('test')
}.each { job ->
    job.disabled = true
    job.save()
    println "Disabled job: ${job.name}"
}

// Get build history
def job = Jenkins.instance.getJob('my-job')
job.builds.each { build ->
    println "Build ${build.number}: ${build.result} (${build.timestampString})"
}

// Clean up old builds (keep last 10)
def job = Jenkins.instance.getJob('my-job')
def builds = job.builds.toList()
if (builds.size() > 10) {
    builds.drop(10).each { build ->
        println "Deleting build ${build.number}"
        build.delete()
    }
}

// Get node information
Jenkins.instance.nodes.each { node ->
    println "Node: ${node.name}"
    println "  Labels: ${node.labelString}"
    println "  Online: ${node.computer.online}"
    println "  Executors: ${node.numExecutors}"
}

// Update plugin
def pluginManager = Jenkins.instance.pluginManager
def plugin = pluginManager.getPlugin('git')
if (plugin && plugin.hasUpdate()) {
    println "Updating Git plugin from ${plugin.version} to ${plugin.updateInfo.version}"
    plugin.deploy(true)
}
```

## Integration with Other Tools

### Docker Integration
```groovy
// Build and push Docker image
pipeline {
    agent any
    
    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    def image = docker.build("myapp:${env.BUILD_NUMBER}")
                    
                    // Run tests in container
                    image.inside {
                        sh 'npm test'
                    }
                    
                    // Push to registry
                    docker.withRegistry('https://registry.company.com', 'registry-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
    }
}

// Docker Compose integration
sh '''
    docker-compose -f docker-compose.test.yml up -d
    sleep 30
    docker-compose -f docker-compose.test.yml exec -T app npm test
    docker-compose -f docker-compose.test.yml down
'''
```

### Kubernetes Integration
```groovy
// Deploy to Kubernetes
pipeline {
    agent any
    
    stages {
        stage('Deploy to K8s') {
            steps {
                script {
                    kubernetesDeploy(
                        configs: 'k8s/*.yaml',
                        kubeconfigId: 'kubeconfig-staging',
                        enableConfigSubstitution: true
                    )
                }
            }
        }
    }
}

// Using kubectl directly
stage('Deploy') {
    steps {
        withKubeConfig([credentialsId: 'kubeconfig-prod']) {
            sh '''
                kubectl apply -f deployment.yaml
                kubectl rollout status deployment/myapp
                kubectl get pods -l app=myapp
            '''
        }
    }
}

// Helm deployment
stage('Deploy with Helm') {
    steps {
        sh '''
            helm upgrade --install myapp ./helm-chart \
                --set image.tag=${BUILD_NUMBER} \
                --set environment=production \
                --namespace=production \
                --wait --timeout=300s
        '''
    }
}
```

### AWS Integration
```groovy
// Deploy to AWS ECS
pipeline {
    agent any
    
    stages {
        stage('Deploy to ECS') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: 'us-west-2') {
                        sh '''
                            aws ecs update-service \
                                --cluster production \
                                --service myapp \
                                --force-new-deployment
                            
                            aws ecs wait services-stable \
                                --cluster production \
                                --services myapp
                        '''
                    }
                }
            }
        }
    }
}

// S3 deployment
stage('Deploy to S3') {
    steps {
        withAWS(credentials: 'aws-credentials', region: 'us-east-1') {
            s3Upload(
                bucket: 'my-static-site',
                path: '',
                includePathPattern: '**/*',
                workingDir: 'dist'
            )
            
            // Invalidate CloudFront cache
            sh '''
                aws cloudfront create-invalidation \
                    --distribution-id E123456789 \
                    --paths "/*"
            '''
        }
    }
}
```

### Terraform Integration
```groovy
// Terraform pipeline
pipeline {
    agent any
    
    stages {
        stage('Terraform Plan') {
            steps {
                withCredentials([azureServicePrincipal('azure-sp')]) {
                    sh '''
                        export ARM_CLIENT_ID=$AZURE_CLIENT_ID
                        export ARM_CLIENT_SECRET=$AZURE_CLIENT_SECRET
                        export ARM_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID
                        export ARM_TENANT_ID=$AZURE_TENANT_ID
                        
                        terraform init
                        terraform plan -out=tfplan
                    '''
                }
            }
        }
        
        stage('Terraform Apply') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Apply Terraform changes?', ok: 'Apply'
                
                withCredentials([azureServicePrincipal('azure-sp')]) {
                    sh '''
                        export ARM_CLIENT_ID=$AZURE_CLIENT_ID
                        export ARM_CLIENT_SECRET=$AZURE_CLIENT_SECRET
                        export ARM_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID
                        export ARM_TENANT_ID=$AZURE_TENANT_ID
                        
                        terraform apply tfplan
                    '''
                }
            }
        }
    }
}
```

## Best Practices and Security

### Security Best Practices
```groovy
// Use credentials securely
pipeline {
    agent any
    
    stages {
        stage('Deploy') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'db-credentials',
                        usernameVariable: 'DB_USER',
                        passwordVariable: 'DB_PASS'
                    ),
                    string(
                        credentialsId: 'api-key',
                        variable: 'API_KEY'
                    )
                ]) {
                    sh '''
                        echo "Connecting to database as $DB_USER"
                        # Never echo passwords!
                        mysql -u "$DB_USER" -p"$DB_PASS" < schema.sql
                    '''
                }
            }
        }
    }
}

// Mask sensitive data
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                script {
                    def sensitiveData = "secret-api-key"
                    // This will be masked in console output
                    wrap([$class: 'MaskPasswordsBuildWrapper', 
                          varPasswordPairs: [[var: 'API_KEY', password: sensitiveData]]]) {
                        sh 'echo "API Key: $API_KEY"'
                    }
                }
            }
        }
    }
}

// Validate inputs
pipeline {
    agent any
    
    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch to build')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target environment')
    }
    
    stages {
        stage('Validate') {
            steps {
                script {
                    // Validate branch name
                    if (!params.BRANCH_NAME.matches(/^[a-zA-Z0-9_\-\/]+$/)) {
                        error("Invalid branch name: ${params.BRANCH_NAME}")
                    }
                    
                    // Validate environment
                    def allowedEnvs = ['dev', 'staging', 'prod']
                    if (!allowedEnvs.contains(params.ENVIRONMENT)) {
                        error("Invalid environment: ${params.ENVIRONMENT}")
                    }
                }
            }
        }
    }
}
```

### Performance Optimization
```groovy
// Parallel execution
pipeline {
    agent none
    
    stages {
        stage('Parallel Tests') {
            parallel {
                stage('Unit Tests') {
                    agent { label 'linux' }
                    steps {
                        sh 'npm run test:unit'
                    }
                }
                stage('Integration Tests') {
                    agent { label 'linux' }
                    steps {
                        sh 'npm run test:integration'
                    }
                }
                stage('Security Scan') {
                    agent { label 'security' }
                    steps {
                        sh 'npm audit'
                    }
                }
            }
        }
    }
}

// Resource management
pipeline {
    agent {
        docker {
            image 'node:16'
            args '--cpus="2" --memory="4g"'
        }
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        retry(3)
        skipDefaultCheckout(true)
    }
    
    stages {
        stage('Build') {
            steps {
                // Efficient checkout
                checkout scm
                
                // Use build cache
                script {
                    if (fileExists('.npm-cache')) {
                        sh 'cp -r .npm-cache ~/.npm'
                    }
                }
                
                sh 'npm ci'
                
                // Save cache
                sh 'cp -r ~/.npm .npm-cache'
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
```

### Code Quality and Testing
```groovy
// Quality gates
pipeline {
    agent any
    
    stages {
        stage('Quality Checks') {
            parallel {
                stage('Code Coverage') {
                    steps {
                        sh 'npm run test:coverage'
                        publishCoverageReports(
                            adapters: [
                                istanbulCoberturaAdapter('coverage/cobertura-coverage.xml')
                            ],
                            sourceFileResolver: sourceFiles('STORE_LAST_BUILD')
                        )
                        
                        script {
                            def coverage = readFile('coverage/coverage-summary.json')
                            def json = new groovy.json.JsonSlurper().parseText(coverage)
                            def linesCoverage = json.total.lines.pct
                            
                            if (linesCoverage < 80) {
                                unstable("Code coverage below threshold: ${linesCoverage}%")
                            }
                        }
                    }
                }
                
                stage('Code Quality') {
                    steps {
                        sh 'npm run lint'
                        recordIssues(
                            enabledForFailure: true,
                            aggregatingResults: true,
                            tools: [
                                esLint(pattern: 'reports/eslint.xml')
                            ]
                        )
                    }
                }
                
                stage('Security Scan') {
                    steps {
                        sh 'npm audit --audit-level moderate --json > audit-report.json || true'
                        
                        script {
                            def auditReport = readJSON file: 'audit-report.json'
                            if (auditReport.metadata.vulnerabilities.high > 0) {
                                error("High severity vulnerabilities found")
                            }
                            if (auditReport.metadata.vulnerabilities.moderate > 5) {
                                unstable("Too many moderate vulnerabilities")
                            }
                        }
                    }
                }
            }
        }
    }
}
```

## Troubleshooting Tips

### Common Issues and Solutions
```groovy
// Handle flaky tests
stage('Tests with Retry') {
    steps {
        retry(3) {
            sh 'npm test'
        }
    }
}

// Timeout handling
stage('Long Running Task') {
    steps {
        timeout(time: 10, unit: 'MINUTES') {
            sh './long-running-script.sh'
        }
    }
}

// Error handling and recovery
pipeline {
    agent any
    
    stages {
        stage('Deploy') {
            steps {
                script {
                    try {
                        sh './deploy.sh'
                    } catch (Exception e) {
                        echo "Deployment failed: ${e.getMessage()}"
                        
                        // Attempt rollback
                        try {
                            sh './rollback.sh'
                            echo "Rollback completed successfully"
                            currentBuild.result = 'UNSTABLE'
                        } catch (Exception rollbackError) {
                            echo "Rollback failed: ${rollbackError.getMessage()}"
                            currentBuild.result = 'FAILURE'
                            throw e
                        }
                    }
                }
            }
        }
    }
}

// Debug information
stage('Debug Info') {
    steps {
        script {
            echo "Jenkins Version: ${Jenkins.instance.version}"
            echo "Java Version: ${System.getProperty('java.version')}"
            echo "Node Name: ${env.NODE_NAME}"
            echo "Workspace: ${env.WORKSPACE}"
            
            // Environment variables
            sh 'printenv | sort'
            
            // System information
            sh '''
                echo "=== System Info ==="
                uname -a
                echo "=== Disk Space ==="
                df -h
                echo "=== Memory ==="
                free -h
                echo "=== Docker Info ==="
                docker info || echo "Docker not available"
            '''
        }
    }
}
```

### Monitoring and Logging
```groovy
// Structured logging
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                script {
                    def startTime = System.currentTimeMillis()
                    
                    try {
                        echo "[INFO] Starting build at ${new Date()}"
                        sh 'npm run build'
                        
                        def endTime = System.currentTimeMillis()
                        def duration = (endTime - startTime) / 1000
                        echo "[INFO] Build completed in ${duration} seconds"
                        
                    } catch (Exception e) {
                        echo "[ERROR] Build failed: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
    }
}

// Performance metrics
stage('Performance Metrics') {
    steps {
        script {
            def buildMetrics = [
                buildNumber: env.BUILD_NUMBER,
                startTime: currentBuild.startTimeInMillis,
                duration: currentBuild.duration ?: 0,
                result: currentBuild.result ?: 'SUCCESS'
            ]
            
            writeJSON file: 'build-metrics.json', json: buildMetrics
            archiveArtifacts artifacts: 'build-metrics.json'
            
            // Send to monitoring system
            httpRequest(
                httpMode: 'POST',
                url: 'https://monitoring.company.com/api/jenkins-metrics',
                contentType: 'APPLICATION_JSON',
                requestBody: groovy.json.JsonOutput.toJson(buildMetrics)
            )
        }
    }
}
```

### Workspace and Build Management
```bash
# Clean workspace
# Configure in job: Build Environment > Delete workspace before build starts

# Or in pipeline:
pipeline {
    agent any
    
    options {
        skipDefaultCheckout(true)
    }
    
    stages {
        stage('Cleanup and Checkout') {
            steps {
                cleanWs()
                checkout scm
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}

# Manage build history
# Configure in job: Discard old builds
# Keep builds for 30 days, maximum 50 builds

# In pipeline:
properties([
    buildDiscarder(logRotator(
        daysToKeepStr: '30',
        numToKeepStr: '50',
        artifactDaysToKeepStr: '7',
        artifactNumToKeepStr: '10'
    ))
])
```

## Official Documentation Links

- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Jenkins Plugins Index](https://plugins.jenkins.io/)
- [Jenkins Blue Ocean](https://www.jenkins.io/doc/book/blueocean/)
- [Jenkins Security](https://www.jenkins.io/doc/book/security/)
- [Jenkins Administration](https://www.jenkins.io/doc/book/system-administration/)
- [Groovy Documentation](https://groovy-lang.org/documentation.html)
- [Jenkins Best Practices](https://www.jenkins.io/doc/book/pipeline/pipeline-best-practices/)