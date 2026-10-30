# Azure DevOps YAML Pipeline for SAP CAP CDS Application Deployment to SAP Cloud Foundry
# Based on original guide: Build a CI/CD Pipeline in Azure DevOps to SAP Cloud Foundry

trigger:
  branches:
    include:
      - master
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  # Artifact Staging Directory for MTA build outputs
  buildOutputDir: '$(Build.ArtifactStagingDirectory)'
  # Set your CAP MTA application archive name (e.g. my-cap-service_0.0.1.mtar)
  mtaFileName: 'cap-service_0.0.1.mtar'
  # Set your Pipeline Artifact name
  artifactName: 'CapServiceArtifact'

stages:
# ==============================================================================
# STAGE 1: BUILD & PACKAGE MTA
# ==============================================================================
- stage: Build
  displayName: 'Build SAP CAP MTA Service'
  jobs:
  - job: BuildJob
    displayName: 'Build & Publish MTA Artifact'
    steps:
    - task: NodeTool@0
      inputs:
        versionSpec: '18.x'
      displayName: 'Install Node.js'

    - script: |
        npm install -g @sap/cds-dk @sap/mbt
      displayName: 'Install Dependencies (@sap/cds-dk & @sap/mbt)'

    - script: |
        npm ci
      displayName: 'Install Project Dependencies'

    - script: |
        mbt build -p=cf -s=. -t=$(buildOutputDir)
      displayName: 'MTA Build'

    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: '$(buildOutputDir)/$(mtaFileName)'
        artifact: '$(artifactName)'
        publishLocation: 'pipeline'
      displayName: 'Publish Pipeline Artifact'

# ==============================================================================
# STAGE 2: DEPLOY TO DEV SPACE
# ==============================================================================
- stage: Dev
  displayName: 'Deploy to Dev Space'
  dependsOn: Build
  jobs:
  - deployment: DeployDev
    displayName: 'Deploy to SAP CF Dev Space'
    environment: 'dev'
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: '$(artifactName)'
            displayName: 'Download MTA Artifact'

          - script: |
              cf install-plugin multiapps -f
            displayName: 'Install Dependency (multiapps plugin)'

          - script: |
              cf login -a $(api-url) -u $(dev-user) -p $(dev-password) -o $(subaccount-id) -s $(dev-org)
            displayName: 'Login to SAP Cloud Foundry (Dev)'

          - script: |
              cd $(Pipeline.Workspace)/$(artifactName)
              cf deploy $(mtaFileName) -f
            displayName: 'Deploy Service to Dev Space'

# ==============================================================================
# STAGE 3: DEPLOY TO PRODUCTION SPACE
# ==============================================================================
- stage: Prd
  displayName: 'Deploy to Productive Space'
  dependsOn: Dev
  jobs:
  - deployment: DeployPrd
    displayName: 'Deploy to SAP CF Production Space'
    # Note: Configure Pre-deployment Approvals under Azure DevOps -> Environments -> 'prd'
    environment: 'prd'
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: '$(artifactName)'
            displayName: 'Download MTA Artifact'

          - script: |
              cf install-plugin multiapps -f
            displayName: 'Install Dependency (multiapps plugin)'

          - script: |
              cf login -a $(api-url) -u $(dev-user) -p $(prd-password) -o $(subaccount-id) -s $(prd-org)
            displayName: 'Login to SAP Cloud Foundry (Prd)'

          - script: |
              cd $(Pipeline.Workspace)/$(artifactName)
              cf deploy $(mtaFileName) -f
            displayName: 'Deploy Service to Production Space'