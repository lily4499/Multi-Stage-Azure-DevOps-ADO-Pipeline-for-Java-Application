# Multi-Stage-Azure-DevOps-ADO-Pipeline-for-Java-Application

```markdown
# 🚀 Multi-Stage Azure DevOps (ADO) Pipeline for Java Application

This guide walks you through creating a multi-stage CI/CD pipeline in **Azure DevOps (ADO)** for a **Java application**, including code validation, integration testing, and deployment to **Azure App Service** or **Azure Kubernetes Service (AKS)**.

---

## 🎯 Project Objective

Create a CI/CD pipeline in Azure DevOps that:

- ✅ Validates Java code with Maven  
- ✅ Runs integration tests  
- ✅ Deploys the app to Azure App Service or AKS  
- ✅ Uses environment approvals before production  

---

## 📁 Project Structure (Java App Example)

```bash
java-app/
├── src/
│   ├── main/java/com/example/demo/App.java
│   └── test/java/com/example/demo/AppTest.java
├── manifests/
│   ├── deployment.yaml
│   └── service.yaml
├── pom.xml
├── Dockerfile      # Required if deploying to AKS
└── azure-pipelines.yml
```
---

##  file-setup.py

```python
import os

# Base directory
base_dir = "/home/lilia/VIDEOS/java-ado-pipeline"
os.makedirs(base_dir, exist_ok=True)

# File definitions with content
files = {
    "pom.xml": """<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>java-ado-pipeline</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>
    <name>Java ADO Pipeline</name>
    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
""",
    "Dockerfile": """FROM openjdk:17-jdk-slim
COPY target/java-ado-pipeline.war /usr/local/tomcat/webapps/
CMD ["catalina.sh", "run"]
""",
    "azure-pipelines.yml": """stages:
- stage: Build
  displayName: "Build & Validate"
  jobs:
  - job: MavenBuild
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: Maven@3
      inputs:
        mavenPomFile: 'pom.xml'
        goals: 'clean compile'

- stage: Test
  displayName: "Integration Tests"
  dependsOn: Build
  jobs:
  - job: RunTests
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: Maven@3
      inputs:
        mavenPomFile: 'pom.xml'
        goals: 'verify'

- stage: DeployAppService
  displayName: "Deploy to Azure App Service"
  dependsOn: Test
  condition: succeeded()
  jobs:
  - deployment: DeployWeb
    environment: 'staging'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'MyAzureServiceConnection'
              appType: 'webApp'
              appName: 'my-java-app'
              package: '$(System.DefaultWorkingDirectory)/**/*.war'

- stage: DeployAKS
  displayName: "Deploy to AKS"
  dependsOn: Test
  condition: succeeded()
  jobs:
  - deployment: DeployToK8s
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: Kubernetes@1
            inputs:
              connectionType: 'Kubernetes Service Connection'
              kubernetesServiceEndpoint: 'MyK8sConnection'
              namespace: 'default'
              command: 'apply'
              useConfigurationFile: true
              configuration: 'manifests/deployment.yaml'
""",
    "manifests/deployment.yaml": """apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: java-app
  template:
    metadata:
      labels:
        app: java-app
    spec:
      containers:
      - name: java-container
        image: laly9999/java-ado-pipeline:latest
        ports:
        - containerPort: 8080
""",
    "manifests/service.yaml": """apiVersion: v1
kind: Service
metadata:
  name: java-app-service
spec:
  type: LoadBalancer
  selector:
    app: java-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
""",
    "src/main/java/com/example/demo/App.java": """package com.example.demo;

public class App {
    public static void main(String[] args) {
        System.out.println("Hello from Java ADO Pipeline!");
    }
}
""",
    "src/test/java/com/example/demo/AppTest.java": """package com.example.demo;

import org.junit.Test;
import static org.junit.Assert.*;

public class AppTest {
    @Test
    public void testMain() {
        assertEquals(1, 1);
    }
}
"""
}

# Create files
for path, content in files.items():
    full_path = os.path.join(base_dir, path)
    os.makedirs(os.path.dirname(full_path), exist_ok=True)
    with open(full_path, 'w') as f:
        f.write(content)

files_created = list(files.keys())
files_created


```

---

## 🛠️ Prerequisites

1. Azure DevOps Project
   
```plain text
You need an Azure DevOps organization and a project.
To create one:
Go to https://dev.azure.com
Click New Project
Name it (e.g., java-ado-pipeline), select visibility (private/public), and create.
This project will host your repos, pipelines, environments, and service connections.
```
```bash
az extension add --name azure-devops
az login
az devops configure --defaults organization=https://dev.azure.com/my-org-name
az devops project create --name java-ado-pipeline
az devops project list
az devops configure --defaults project=java-ado-pipeline

```


2. Azure Subscription linked to ADO
```bash
#Login to Azure & Set Subscription
az login
az account set --subscription "<Your-Subscription-Name-or-ID>"
az account show --output table

#Create a Service Principal (SP)
az ad sp create-for-rbac \
  --name "ado-sp-java-app" \
  --role contributor \
  --scopes /subscriptions/<your-subscription-id> \
  --sdk-auth > ado-sp-java-app.json

#Create the Service Connection in Azure DevOps (Manual Step)
  Go to Azure DevOps → Project Settings → Service Connections
  Click New service connection
  Choose Azure Resource Manager
  Select Service Principal (manual)
  Enter:
      Subscription ID: subscriptionId from JSON
      Subscription Name: from az account show
      Tenant ID: tenantId
      Service Principal ID: clientId
      Service Principal Key: clientSecret
  Verify and Save ✅

```


3. Java codebase with `pom.xml`
 - A Java application managed with Maven is required.
 - pom.xml should contain at minimum:
    Java version
    Build plugins
    JUnit test dependencies
 
   
4. App Service or AKS set up in Azure
```bash
 Option 1: Azure App Service Setup (Simple Web App Hosting)
✅ Use Case:
Best for simple web apps (e.g., Spring Boot or Java WAR/JAR) where you want fast deployment and don’t need container orchestration.
#Create a Resource Group
az group create --name java-app-rg --location eastus

#Create an App Service Plan
az appservice plan create \
  --name java-app-plan \
  --resource-group java-app-rg \
  --sku B1 \
  --is-linux

#Create the Web App (Linux, Java 11)
az webapp create \
  --resource-group java-app-rg \
  --plan java-app-plan \
  --name java-javaweb123 \
  --runtime "JAVA|11-java11" \
  --deployment-container-image-name laly9999/java-ado-app:latest

#Deploy the app: If using WAR/JAR:
az webapp deploy \
  --resource-group java-app-rg \
  --name java-javaweb123 \
  --src-path target/demo.jar \
  --type jar
```

---

```bash
Option 2: Azure Kubernetes Service (AKS) Setup
✅ Use Case:
Best for microservices or containerized apps that need scalability, orchestration, or advanced networking.
#Create Resource Group
az group create --name java-aks-rg --location eastus

#Create AKS Cluster
az aks create \
  --resource-group java-aks-rg \
  --name javaAksCluster \
  --node-count 2 \
  --enable-addons monitoring \
  --generate-ssh-keys

# Get AKS Credentials
az aks get-credentials --resource-group java-aks-rg --name javaAksCluster

# Deploy your Java App (using Kubernetes YAML)

```
---


5. Service Connection in ADO:
  -  Azure Resource Manager for App Service
  -  Kubernetes Service Connection for AKS


```bash
1️⃣ Create Azure Resource Manager (ARM) Service Connection
For deploying to Azure App Service
#Get Required Details
az login
az account show --query "{subscriptionId:id, tenantId:tenantId}" --output json

#Create a Service Principal (if not done)

#Create Azure Resource Manager Service Connection via REST API
#Set Azure DevOps organization and project
ORG_URL="https://dev.azure.com/<your-org-name>"
PROJECT_NAME="<your-project-name>"
#Get Personal Access Token (PAT)
Create a PAT from: https://dev.azure.com → User Settings → Personal Access Tokens
Scopes: Service Connections (Read & Manage), Project & Team (Read)
#. Encode PAT
PAT="<your-pat>"
AUTH_HEADER="Authorization: Basic $(echo -n :$PAT | base64)"
# Run REST call:
curl -X POST "$ORG_URL/$PROJECT_NAME/_apis/serviceendpoint/endpoints?api-version=7.0-preview.4" \
-H "Content-Type: application/json" \
-H "$AUTH_HEADER" \
-d '{
  "name": "arm-appservice-conn",
  "type": "azurerm",
  "authorization": {
    "scheme": "ServicePrincipal",
    "parameters": {
      "tenantid": "<tenantId>",
      "serviceprincipalid": "<clientId>",
      "authenticationType": "spnKey",
      "serviceprincipalkey": "<clientSecret>"
    }
  },
  "data": {
    "subscriptionId": "<subscriptionId>",
    "subscriptionName": "My Azure Sub",
    "environment": "AzureCloud",
    "scopeLevel": "Subscription"
  },
  "isShared": false,
  "isReady": true
}'

```
---

```bash
2️⃣ Create Kubernetes Service Connection for AKS
#Get kubeconfig credentials
az aks get-credentials --resource-group <rg-name> --name <aks-name> --file aks-kubeconfig
#Encode the kubeconfig
KUBECONFIG_BASE64=$(base64 -w 0 < aks-kubeconfig)
#REST API to Create Kubernetes Service Connection
curl -X POST "$ORG_URL/$PROJECT_NAME/_apis/serviceendpoint/endpoints?api-version=7.0-preview.4" \
-H "Content-Type: application/json" \
-H "$AUTH_HEADER" \
-d '{
  "name": "aks-service-conn",
  "type": "kubernetes",
  "authorization": {
    "scheme": "None"
  },
  "data": {
    "authorizationType": "Kubeconfig",
    "kubeconfig": "'$KUBECONFIG_BASE64'"
  },
  "isReady": true
}'

```

---
---

## 🧱 Step-by-Step Pipeline Breakdown

### 🧩 Stage 1: Code Validation

```yaml
stages:
- stage: Build
  displayName: "Build & Validate"
  jobs:
  - job: MavenBuild
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: Maven@3
      inputs:
        mavenPomFile: 'pom.xml'
        goals: 'clean compile'
        mavenOptions: '-Xmx1024m'
```

---

### 🧪 Stage 2: Integration Testing

```yaml
- stage: Test
  displayName: "Integration Tests"
  dependsOn: Build
  jobs:
  - job: RunTests
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: Maven@3
      inputs:
        mavenPomFile: 'pom.xml'
        goals: 'verify'
```

---

### 🚀 Stage 3: Deploy to Azure App Service

```yaml
- stage: DeployAppService
  displayName: "Deploy to Azure App Service"
  dependsOn: Test
  condition: succeeded()
  jobs:
  - deployment: DeployWeb
    environment: 'staging'  # Use approvals in ADO environment settings
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'MyAzureServiceConnection'
              appType: 'webApp'
              appName: 'my-java-app'
              package: '$(System.DefaultWorkingDirectory)/**/*.war'
```

---

### ☸️ Optional: Deploy to AKS

```yaml
- stage: DeployAKS
  displayName: "Deploy to AKS"
  dependsOn: Test
  condition: succeeded()
  jobs:
  - deployment: DeployToK8s
    environment: 'production'  # Add manual approval in Azure DevOps UI
    strategy:
      runOnce:
        deploy:
          steps:
          - task: Kubernetes@1
            inputs:
              connectionType: 'Kubernetes Service Connection'
              kubernetesServiceEndpoint: 'MyK8sConnection'
              namespace: 'default'
              command: 'apply'
              useConfigurationFile: true
              configuration: 'manifests/deployment.yaml'
```

---

## 🔐 Set Up Approvals

- Go to **Pipelines > Environments**
- Select `staging` and `production`
- Click **Approvals and Checks**
- Add manual review or branch protections as needed

---

## 📝 Final Tips

- Use `mvn package` or `mvn verify` depending on how your tests are structured
- For **App Service**, deploy `.war` or `.jar` file
- For **AKS**, ensure `deployment.yaml` and `service.yaml` are present in the `manifests/` directory

---

> Created by [Liliane Konissi](https://github.com/lily4499) 💡 | DevOps | Azure | CI/CD
```

