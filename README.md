# CICD_Automation_for_Azure_Data_Factory

This project implements **CI/CD automation** for Azure Data Factory using  **Azure DevOps**, enabling consistent, reliable, and fully automated deployment of ADF pipelines — including components such as Linked Services, Datasets, and Triggers — across **Development**, **UAT**, and **Production** environments. 

### Key Benefits:

- **Reduces manual intervention and errors**, ensuring higher reliability and fewer production issues 
- **Improves deployment efficiency** through automation of build and release workflows
- **Ensures environment consistency** by applying the same artifacts and templates across all stages
- **Accelerates delivery cycles** by shortening release timelines and automating validations

---

# 📘 Introduction

**Azure Data Factory (ADF)** is a service in Microsoft Azure that allows developers to integrate and transform data using various processes. ADF provides strong integration with multiple internal and external services.

In ADF, **CI/CD** essentially means deploying Data Factory components (**Linked Services**, **Triggers**, **Pipelines**, **Datasets**) to new environments automatically using pipelines.

---

# ADF Modes

ADF operates in two modes:

- **Live Mode**
- **Git Mode**

When Git integration is enabled, ADF requires the selection of two branches:

- **Collaboration Branch** – where all feature branches are merged (mapped to `develop` in our case)
- **Publish Branch** – where all changes and auto-generated ARM templates are published (default: `adf_publish`)

A corresponding `adf` folder in the repository contains all ADF resource definitions.

---

# Deployment Method

There are two suggested methods to promote a data factory to another environment:

#### 1. Manually upload a Resource Manager template using Data Factory UX integration with Azure Resource Manager – Manual Publish

Previously, ADF deployment relied on **manual publishing** from the collaboration branch. This action generated ARM templates and stored them in the `adf_publish` branch. The deployment was then performed using these templates.

#### 🔴 Drawbacks:
- Developers had to manually click the **Publish** button in the ADF portal.
- ARM templates were generated only after this manual action.


#### 2. Automated deployment using Data Factory’s integration with Azure Pipelines - Automatic Publish

In this method, the **manual publish step is eliminated**. Instead, **automatic publishing** is implemented using the **Build Pipeline** and an NPM package to generate ARM templates as artifacts.

---

# Architecture

Below is the architecture of the ADF CI/CD implementation:

![ADF_CICD_Architecture](/ADF_CICD_Architecture.jpg)

In the diagram above, there is an **ADF instance in the Dev environment** linked to **GitHub**. Development activities are carried out on the Dev ADF using **feature branches**, and changes are merged into the **`develop` branch** via **pull requests**. Once merged, the updates are deployed to the Dev ADF.

After the merge, the **ADF CI/CD pipeline** is triggered, which automatically deploys the ADF resources to the **Dev** and **Prod** environments.

---

## 📁 Project Structure


📁 build/                            # 🔧 Build folder

    ├── adf-pipeline.yml                 # Main Azure DevOps YAML pipeline (Build + Deploy)

    └── package.json                     # Defines npm build/export/validate scripts

📁 data_ops/adf/                          # Core ADF source code folder

    ├── 📁 dataset/

        ├── source.json              # Dataset definition for source

        └── destination.json         # Dataset definition for destination

    ├── 📁 linkedService/

        ├── adls_connection.json     # Linked Service: Azure Data Lake
            
        ├── databricks_connection.json # Linked Service: Azure Databricks
            
        └── keyvault_connection.json # Linked Service: Azure Key Vault

    ├── 📁 pipeline/
     
        ├── copy_data.json           # Pipeline definition 1
            
        └── pipeline1.json           # Pipeline definition 2

    └── 📁 trigger/
    
        └── trigger1.json            # Trigger for pipeline scheduling

├── ADF_CICD_Architecture.jpg            # Architecture diagram for documentation/reference

├── README.md                            # 📘 Project document


---

# Getting start

## ✅ Prerequisites

Before implementing CI/CD for Azure Data Factory using Azure DevOps, ensure the following components are set up and correctly configured:

### 🔹 **Azure Data Factory (ADF) Instances**
You should have separate ADF instances for each environment:
- **Dev** – where development and testing in Git mode happens.
- **Test/Pre-Prod** – staging environment for testing release workflows.
- **Prod** – final deployment environment for live data workflows.

> 💡 ADF in Dev should be linked with Git repo for collaboration. Deployments to Test and Prod are done using ARM templates from Dev’s *live mode*.


### 🔹 **Azure DevOps Project**
Ensure you have an Azure DevOps project set up, which includes:
- **Repos** – to store the ADF ARM templates and YAML pipeline code.
- **Pipelines** – for defining build (export) and release (deployment) workflows.

### 🔹 **Service Principal (SP)**
A **Service Principal** (App Registration) is required to authenticate Azure DevOps to access Azure resources.

#### ✅ Option A: Create Service Principal Manually

##### Steps:
1. **Register an App**:  
   Go to Azure AD → *App registrations* → **New registration**  
   Name: `azure-devops-sp` | Supported account type: **Single tenant**

2. **Create a Client Secret**:  
   Go to *Certificates & secrets* → **New client secret**  
   Copy the secret **value** and store it securely.

3. **Collect Required Values**:
   - `Tenant ID` → Azure AD → Overview
   - `Client ID` → App registration overview
   - `Client Secret` → From above
   - `Subscription ID` → Azure > Subscriptions

4. **Assign Role** to the SP:
   Go to Subscription → IAM → Add role assignment  
   Role: **Contributor**  
   Assign to: Service Principal created above

> Use this SP in a manual Azure DevOps service connection.


### 🔹 **Service Connection in Azure DevOps**

Azure DevOps needs a **Service Connection** to authenticate and deploy ADF resources to Azure. You can configure this in two ways:

#### ✅ Option A: Manual (using existing SP)
> Recommended when fine-grained control or least privilege principle is needed.

##### 📌 Steps:
1. Azure DevOps → *Project Settings* → *Service Connections* → **New**
2. Select: **Azure Resource Manager**
3. Choose: **Service Principal (manual)**
4. Enter:
   - Subscription ID
   - Subscription Name
   - Client ID (SP ID)
   - Client Secret
   - Tenant ID
   - Environment: `AzureCloud`
5. Click **Verify and Save**


#### ✅ Option B: Automatic (with auto-created SP)
> Recommended for quick setup. You must be **Owner** on the subscription.

##### 📌 Steps:
1. Azure DevOps → *Project Settings* → *Service Connections* → **New**
2. Select: **Azure Resource Manager**
3. Choose: **Service Principal (automatic)** (Recommended)
4. Select:
   - Scope level: **Subscription**
   - Subscription: Choose from list
   - (Optional) Resource Group: For narrower access
5. Provide a name (e.g., `AzureSP-Connection`)
6. Click **Verify and Save**


### 🔹 Access to External Azure Resources (Linked Services)

To enable **Azure Data Factory (ADF)** to interact with services like **Azure Storage**, **Azure Synapse**, and **Azure Key Vault**, you must assign appropriate **RBAC roles** to its **Managed Identity**.

> ⚠️ Each ADF environment (**Dev**, **UAT**, **Prod**) has a **unique System Assigned Managed Identity**. Roles must be granted **per environment**.

#### Common Azure Resources Accessed via Linked Services

- **Azure Data Lake / Blob Storage**  
- **Azure Synapse Analytics**  
- **Azure SQL Database**  
- **Azure Key Vault**  

#### 📌 Example: Granting Access to Azure Storage Account

To allow ADF to read/write to Azure Blob Storage:

1. Go to your **Storage Account** ➝ **Access Control (IAM)**
2. Click **➕ Add role assignment**
3. Select the **Role**: `Storage Blob Data Contributor`
4. Assign access to: **Managed Identity**
5. Select your **ADF instance**
6. Click **Review + assign**

> Repeat similar steps for Synapse, SQL DB, and Key Vault, with their respective roles.

#### 🔁 Repeat Per Environment: Dev, UAT, and Prod

Each environment has its own ADF instance and **unique Managed Identity**.  
You must assign the above roles **individually** in each environment.


#### 🔄 What You Need To Do for Each Environment (UAT / Prod)

Ensure secure access for **ADF in UAT and Production** environments by following these steps:

#### Step-by-Step Instructions

1. **Enable Managed Identity** (if not already enabled):  
   - Go to **ADF (UAT/Prod)** ➝ **Identity** under **Settings**  
   - Turn **System Assigned Identity** to **On**  
   - Click **Save**

2. **Copy the Object ID** of the Managed Identity (needed for role assignment)

3. **Grant Access via IAM or Access Policies** on the following resources:

   - 🔹 **Azure Storage Account**
   - 🔹 **Azure Synapse Analytics**
   - 🔹 **Azure Key Vault**
   - 🔹 **Any additional resources** used in ADF pipelines

4. **Match Role Assignments from Dev**:  
   Ensure the same **RBAC roles or Access Policy configurations** used in Dev are applied to UAT and Prod.

#### ✅ Summary

Always ensure that ADF’s **Managed Identity** has the appropriate **RBAC roles** or **Access Policies** for every **Linked Service** used in the environment.

> 🔑 This step is **critical** to ensure secure and successful pipeline executions across **all environments**.


### 🔹 6. Git Integration in ADF (Dev Environment Only)

Git integration in Azure Data Factory (ADF) is essential for **version control**, **collaboration**, and enabling **CI/CD workflows**. This setup is typically performed only in the **Development environment**.

### 🔧 Setup Instructions

1. Go to **ADF Studio** ➝ Click on **Manage** (gear icon).
2. Select **Git Configuration** under the **Source control** section.
3. Choose **Azure DevOps Git** as the Git repository type.
4. Link your Azure DevOps repository and specify the following:

   - **Collaboration branch**:  
     Typically set to `develop`

   - **Root folder**:  
     The path within the repository where ADF files will reside

   - **Publish branch**:  
     Set to `adf_publish` (automatically used by ADF to store ARM templates for deployment)

---

## Development Flow for ADF

1. In the ADF portal, create a **feature branch** (e.g., `feature_1`) from your **collaboration branch** (`develop`).
2. Develop and **manually test** your changes within the feature branch in the ADF portal.
3. Create a **Pull Request (PR)** from the feature branch to the collaboration branch in GitHub.
4. Once the PR is merged, the changes will be **deployed to the ADF Dev environment**.

---

## Adding the `package.json`

Before setting up the build pipeline, a `package.json` file needs to be created. This file is required to fetch the **ADFUtilities** NPM package used in the CI/CD process.

### Steps to Add `package.json`:

1. In your repository, create a folder named `build` (you can choose any name).
2. Inside the folder, create a file named `package.json`.
3. Paste the following content into the `package.json` file:

```json
{
    "scripts":{
        "build":"node node_modules/@microsoft/azure-data-factory-utilities/lib/index"
    },
    "dependencies":{
        "@microsoft/azure-data-factory-utilities":"^0.1.5"
    }
}
```

---

## Creating the Azure YAML Pipeline

The Azure YAML pipeline file defines **CI (Continuous Integration)** and **CD (Continuous Deployment)** stages, each with its respective set of tasks.

### 🔹 `trigger` block
The pipeline begins with a **`trigger`** block. This defines when the pipeline should automatically run.

```yaml
trigger:
  batch: false
  branches:
    include:
      - develop
      - main
```

`batch: false`:
   - When set to false, the pipeline will trigger immediately for each new commit to a branch.
   - If it were true, it would batch multiple commits into a single run, avoiding multiple pipeline executions for closely timed commits.

`branches:include`:
   - This list specifies the branches that should trigger the pipeline.
   - In this case, the pipeline runs on any push to the develop or main branches.
   - This includes PR merges or direct commits.


### 🔹 `pool` block
This specifies the build agent (host machine) for running the pipeline jobs.

```yaml
pool:
  vmImage: 'ubuntu-latest'
```

   - It uses a Microsoft-hosted Ubuntu virtual machine image with common tools pre-installed.

   - Suitable for most build/deployment tasks including Node.js, Python, .NET, Azure CLI, etc.


### 🔹 `variables` block

In this setup, variables are stored in **Azure DevOps Variable Groups** for better modularity and security.

In our example, we have two variable groups corresponding to the **Dev** and **Prod** environments.

Create Variable Groups in Azure DevOps and Reference Them Conditionally in Your YAML Pipeline.

- Go to your **Azure DevOps project**.
- Navigate to: **Pipelines** → **Library**
- Click **+ Variable group**
- Create one group called `adf-dev` and another called `adf-prod`
- Add required variables (see below).
- Save both variable groups.

> 🔐 If you're using secret values (e.g., passwords or connection strings), mark them as **secrets**.

#### 📌 Required Variables

The following variables are required within the variable groups:

- `azure_subscription_id` – Your Azure subscription ID  
- `azure_service_connection_name` – The name of the Azure DevOps service connection  
- `resource_group_name` – The name of the resource group  
- `azure_data_factory_name` – The name of the ADF instance  

In addition to these, the pipeline defines two more key variables:

- `adf_code_path` – Path in the repo where ADF code is stored  
  > This should match the **Root folder** selected while linking Git with ADF.

- `adf_package_file_path` – Path in the repo where the `package.json` file is located  
  > This is used for generating **ARM templates** using the NPM build utility.

```yaml
variables:

  - ${{ if eq(variables['Build.SourceBranchName'], 'develop') }}:
      - group: adf-dev

  - ${{ if eq(variables['Build.SourceBranchName'], 'main') }}:
      - group: adf-prod

  - name: adf_code_path
    value: "$(Build.SourcesDirectory)/data_ops/adf"

  - name: adf_package_file_path
    value: "$(Build.SourcesDirectory)/build/"
```


### 🔹 Validate ADF (Optional Stage)

This is an **optional stage** in the pipeline where **ADF resources** such as **Linked Services**, **Pipelines**, and **Datasets** stored in the repository are **validated** before deployment.

#### 🛠️ Step Details:

- The stage includes a single step that uses an **Azure DevOps extension** to perform the validation of ADF resources.

```yaml
stages:
  - stage: Validate_ADF_Code
    pool:
      vmImage: 'windows-latest'
    jobs:
    - job: Build_ADF_Code
      displayName: 'Validating the ADF Code'
      steps:
      - task: BuildADFTask@1
        inputs:
          DataFactoryCodePath: '$(adf_code_path)'
          Action: 'Build'
```


### 🔹 CI Process (Build Stage)

In the **Build stage**, the objective is to:

- Validate the ADF code
- Retrieve files from the `develop` branch of the Git repository
- Automatically generate **ARM templates** for the deployment stage

### Build Stage Structure

The Build stage consists of the following **5 steps**:

### 1️⃣ Declare the Build Stage

Create a stage named `Build_And_Publish_ADF_Artifacts` which will contain all the build-related steps:

```yaml
  - stage: Build_And_Publish_ADF_Artifacts
    jobs:
      - job: Build_Adf_Arm_Template
        displayName: 'ADF - ARM template'
        steps:
```

### 2️⃣ Install Dependencies

The next step is to **install the required dependencies**.

Azure provides a tool called **ADFUtilities**, which is used to **validate** ADF code and **generate deployment templates (ARM templates)**.

To use this package, you need to:

- Install **Node.js**
- Use **NPM** (Node Package Manager) to install the ADFUtilities package

```yaml
- task: NodeTool@0
  inputs:
    versionSpec: '10.x'
  displayName: 'Install Node.js'

- task: Npm@1
  inputs:
    command: 'install'
    workingDir: '$(adf_package_file_path)' #replace with the package.json folder.     
    verbose: true
  displayName: 'Install NPM Package'
```

### 3️⃣ Validate ADF Code

The next step is to **validate all Azure Data Factory resource code** in the repository.

This step:

- Calls the `validate` function from the **ADFUtilities** package
- Uses the path where the ADF code is stored in the repository (`adf_code_path`)
- Is executed from the working directory where **ADFUtilities** is installed (`adf_package_file_path`)

> ✅ This validation helps ensure the ADF code is correctly configured before generating ARM templates or deploying.

```yaml
- task: Npm@1
  displayName: 'Validate ADF Code'
  inputs:
    command: 'custom'
    workingDir: '$(adf_package_file_path)' #replace with the package.json folder. 
    customCommand: 'run build validate $(adf_code_path) /subscriptions/$(azure_subscription_id)/resourceGroups/$(resource_group_name)/providers/Microsoft.DataFactory/factories/$(azure_data_factory_name)' 
```

### 4️⃣ Generate ARM Template

The next step is to **generate the ARM template** from the Azure Data Factory source code.

This step:

- Uses the `export` function provided by the **ADFUtilities** package  
- Outputs the generated ARM template into the `ArmTemplate` folder inside the working directory (`adf_package_file_path`)

> 📦 These templates are later used in the deployment (CD) stage to provision ADF resources across environments.

```yaml
- task: Npm@1
  displayName: 'Validate and Generate ARM template'
  inputs:
    command: 'custom'
    workingDir: '$(adf_package_file_path)' #replace with the package.json folder.
    customCommand: 'run build export $(adf_code_path) /subscriptions/$(azure_subscription_id)/resourceGroups/$(resource_group_name)/providers/Microsoft.DataFactory/factories/$(azure_data_factory_name) "ArmTemplate"'  # Change "ADFIntegration" to the name of your root folder, if there is not root folder, remove that part.
```

### 5️⃣ Publish ARM Template as Pipeline Artifact

Finally, the generated **ARM template** is **published as a pipeline artifact**.

- This artifact will be named: `adf-artifact-$(Build.BuildNumber)`
- It can be used in the **Release (CD) stage** to deploy ADF resources to target environments

> Publishing artifacts ensures portability and reusability of deployment assets across stages and pipelines.

```yaml
- task: PublishPipelineArtifact@1
  displayName: Download Build Artifacts - ADF ARM templates
  inputs:
    targetPath: '$(Build.SourcesDirectory)/.pipelines/ArmTemplate' #replace with the package.json folder.
    artifact: 'adf-artifact-$(Build.BuildNumber)'
    publishLocation: 'pipeline'
```

---

### 🔹 CD Process (Deployment Stage)

The main goal of the **Deployment (CD) stage** is to **deploy Azure Data Factory (ADF) resources**.  
This is accomplished by using the **ARM templates** that were generated during the **Build (CI) stage**.


### Deployment Stage Steps

The Deployment stage includes the following steps:


### 1️⃣ Create a New Stage: `Deploy_to_Dev`

Define a stage named `Deploy_to_Dev`. This stage should run **only if** the following conditions are met:

- ✅ The **CI Build stage** has completed successfully  
- ✅ The code changes have been **merged via a Pull Request**  
- ✅ The pipeline was **triggered from the `develop` branch**

> ⚠️ Conditional execution ensures that deployments only happen in controlled and validated scenarios.

```yaml
- stage: Deploy_to_Dev
  condition: and(succeeded(), eq(variables['build.SourceBranchName'], 'develop'))
  displayName: Deploy To Development Environment
  dependsOn: Build_And_Publish_ADF_Artifacts
  jobs: 
    - job: Deploy_Dev
      displayName: 'Deployment - Dev'
      steps:
```

### 2️⃣ Download ARM Template Artifacts

Download the **ARM template artifacts** that were published during the **Build (CI) stage**.

- By default, pipeline artifacts are published in the `$(Pipeline.Workspace)` directory.

> 📁 These artifacts include the deployment templates and parameters needed for provisioning ADF resources in the target environment.


```yaml
- task: DownloadPipelineArtifact@2
  displayName: Download Build Artifacts - ADF ARM templates
  inputs: 
    artifactName: 'adf-artifact-$(Build.BuildNumber)'
    targetPath: '$(Pipeline.Workspace)/adf-artifact-$(Build.BuildNumber)'
```

### 3️⃣ Stop Triggers Before Deployment (Optional but Recommended)

Azure Data Factory can contain **Triggers** that automatically execute pipelines based on a schedule or specific conditions.

> 🔒 It is a **best practice to stop all triggers before deployment** to prevent pipelines from running during the deployment process.

- This step is **optional**, but **highly recommended** to avoid conflicts, partial runs, or data inconsistency during updates.

```yaml
- task: toggle-adf-trigger@2
  displayName: STOP ADF Triggers before Deployment
  inputs:
    azureSubscription: '$(azure_service_connection_name)'
    ResourceGroupName: '$(resource_group_name)'
    DatafactoryName: '$(azure_data_factory_name)'
    TriggerFilter: ''  #Name of the trigger. Leave empty if you want to stop all trigger
    TriggerStatus: 'stop'
```

### 4️⃣ Deploy ARM Templates

This is the step where the **actual deployment** of ADF resources takes place.

- The **ARM templates** generated during the Build stage are deployed from the pipeline artifacts.
- The `overrideParameters` section is used to pass **custom values** (e.g., for Linked Services) to **override default parameters** defined in the templates.

> ⚙️ This enables environment-specific customization while reusing the same templates across Dev, UAT, and Prod.

```yaml
- task: AzureResourceManagerTemplateDeployment@3
  displayName: 'Deploying to Development'
  inputs:
    deploymentScope: 'Resource Group'
    azureResourceManagerConnection: '$(azure_service_connection_name)'
    subscriptionId: '$(azure_subscription_id)'
    action: 'Create Or Update Resource Group'
    resourceGroupName: '$(resource_group_name)'
    location: '$(location)'
    templateLocation: 'Linked artifact'
    csmFile: '$(Pipeline.Workspace)/adf-artifact-$(Build.BuildNumber)/ARMTemplateForFactory.json'
    csmParametersFile: '$(Pipeline.Workspace)/adf-artifact-$(Build.BuildNumber)/ARMTemplateParametersForFactory.json'
    overrideParameters: '-factoryName $(azure_data_factory_name) -adls_connection_properties_typeProperties_url "https://$(azure_storage_account_name).dfs.core.windows.net/" -databricks_connection_properties_typeProperties_existingClusterId $(azure_databricks_cluster_id) -keyvault_connection_properties_typeProperties_baseUrl "https://$(azure_keyvault_name).vault.azure.net/"'
    deploymentMode: 'Incremental'
```

> 💡 **Note:**  
To successfully deploy **Linked Services**, ensure that the **Service Principal** or **Managed Identity** used in the pipeline has **sufficient access** to the Azure resources referenced by ADF — such as **Key Vault**, **Azure Databricks**, or **Storage Accounts**.

---

### 5️⃣ Restart ADF Triggers After Deployment

Once the deployment is complete, **restart the ADF triggers** that were stopped in **Step 3**.

- This ensures that your **pipelines resume their scheduled or event-based executions** as configured.

> 🔁 This step is essential to restore ADF pipeline automation after a successful deployment.

```yaml
- task: toggle-adf-trigger@2
   displayName: START ADF Triggers after Deployment
   inputs:
    azureSubscription: '$(azure_service_connection_name)'
    ResourceGroupName: '$(resource_group_name)'
    DatafactoryName: '$(azure_data_factory_name)'
    TriggerFilter: '' #Name of the trigger. Leave empty if you want to start all trigger
    TriggerStatus: 'start'
```

### 6️⃣ Validate ADF Linked Services (Optional but Recommended)

The final step — while **optional** — can be very **useful** in ensuring the integrity of your deployment.

- **ADF uses Linked Services** to connect with other Azure resources (e.g., Key Vault, Blob Storage, Databricks).
- It's a **best practice** to **validate that all Linked Services are correctly configured and functional** after deployment.

> 🔍 This helps catch any misconfigurations or permission issues early, ensuring smooth operation of your ADF pipelines.

```yaml
- task: TestAdfLinkedServiceTask@1
  displayName: TEST connection of ADF Linked Services
  inputs:
    azureSubscription: '$(azure_service_connection_name)'
    ResourceGroupName: '$(resource_group_name)'
    DataFactoryName: '$(azure_data_factory_name)'
    LinkedServiceName: 'keyvault_connection,adls_connection,databricks_connection' #Comma separated Names of Linked Service whose connection you want to check
    ClientID: '$(service_principal_client_id)'
    ClientSecret: '$(service_principal_client_secret)'
```

> 💡 **Note:** Make sure to define `service_principal_client_id` & `service_principal_client_secret` variables in a Key Vault–linked Variable Group, as we did in the earlier step.


## 🚀 Deployment to a New Environment

In the previous sections, we covered the **development flow** and **deployment process** happening on the **same ADF instance** — typically the one in the **Dev environment**.

- 📌 **Development** occurs in **Git mode**
- ⚙️ **Deployment** is performed in **Live mode**

---

### Deploying to a New Environment (e.g., Prod)

In this section, we’ll look at how to **deploy ADF resources to a completely new environment**, such as **Prod**.

- The deployment process uses the **same ARM template artifacts** generated in the **Build stage**
- The `overrideParameters` section is used to provide **environment-specific values**, such as different Linked Services, Key Vault URIs, Storage Accounts, etc.

> 🧩 This approach promotes reusability and ensures consistent deployments across multiple ADF environments.


```yaml
- stage: Deploy_to_Prod
    condition: and(succeeded(), eq(variables['build.SourceBranchName'], 'main'))
    displayName: Deploy To Production Environment
    dependsOn: Build_And_Publish_ADF_Artifacts
    jobs: 
      - job: Deploy_Prod
        displayName: 'Deployment - Prod'
        steps:
        - task: DownloadPipelineArtifact@2
          displayName: Download Build Artifacts - ADF ARM templates
          inputs: 
            artifactName: 'adf-artifact-$(Build.BuildNumber)'
            targetPath: '$(Pipeline.Workspace)/adf-artifact-$(Build.BuildNumber)'

        - task: toggle-adf-trigger@2
          inputs:
            azureSubscription: '$(azure_service_connection_name)'
            ResourceGroupName: '$(resource_group_name)'
            DatafactoryName: '$(azure_data_factory_name)'
            TriggerFilter: ''  #Name of the trigger. Leave empty if you want to stop all trigger
            TriggerStatus: 'stop'

        - task: AzureResourceManagerTemplateDeployment@3
          displayName: 'Deploying to Production'
          inputs:
            deploymentScope: 'Resource Group'
            azureResourceManagerConnection: '$(azure_service_connection_name)'
            subscriptionId: '$(azure_subscription_id)'
            action: 'Create Or Update Resource Group'
            resourceGroupName: '$(resource_group_name)'
            location: '$(location)'
            templateLocation: 'Linked artifact'
            csmFile: '$(Pipeline.Workspace)/adf-artifact-$(Build.BuildNumber)/ARMTemplateForFactory.json'
            csmParametersFile: '$(Pipeline.Workspace)/adf-artifact-$(Build.BuildNumber)/ARMTemplateParametersForFactory.json'
            overrideParameters: '-factoryName "$(azure_data_factory_name)" -adls_connection_properties_typeProperties_url "https://$(azure_storage_account_name).dfs.core.windows.net/" -databricks_connection_properties_typeProperties_existingClusterId $(azure_databricks_cluster_id) -keyvault_connection_properties_typeProperties_baseUrl "https://$(azure_keyvault_name).vault.azure.net/"'
            deploymentMode: 'Incremental'

        - task: toggle-adf-trigger@2
          inputs:
            azureSubscription: '$(azure_service_connection_name)'
            ResourceGroupName: '$(resource_group_name)'
            DatafactoryName: '$(azure_data_factory_name)'
            TriggerFilter: ''  #Name of the trigger. Leave empty if you want to start all trigger
            TriggerStatus: 'start'
```
> 💡 **Note:** 
Make sure to define the variables used in overrideParameters (like azure_data_factory_name, azure_storage_account_name, azure_databricks_cluster_id, azure_keyvault_name, etc.) for each specific environment (like Dev, Test, Prod) in a Key Vault–linked Variable Group. 

---

## 🌟 Learnings & Highlights

   - Implemented end-to-end CI/CD for Azure Data Factory using Azure DevOps YAML pipelines.

   - Automated ADF deployments across Dev, UAT, and Prod with reusable, parameterized ARM templates.

   - Integrated Git version control with ADF for collaborative development and publish workflows.

   - Secured deployments using Service Principals, Managed Identities, and Key Vault integration.

   - Ensured reliable releases with pre-deployment validation, trigger management, and environment-specific configs.

---

### 🔗 Connect With Me

- [LinkedIn](https://www.linkedin.com/in/ishant-kumar-534989233)

 Tags
#AzureDataFactory #AzureDevOps #CI/CD #ARMTemplates #DataEngineering #InfrastructureAsCode #GitIntegration #KeyVault #ADF #YAML #CloudAutomation #DevOps

