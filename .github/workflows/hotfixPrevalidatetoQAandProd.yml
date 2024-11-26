name: Pre Validate to Staging and PROD  
run-name: ${{ github.actor }} is running the Github Actions 🚀  
  
on:  
  pull_request:  
    types: [opened, edited, reopened, synchronize]  
    branches:  
      - main  
    paths:  
      - 'force-app/**'  
  
jobs:  
  build-and-deploy-Staging:  
    if: contains(github.head_ref, 'hotfix') && github.base_ref == 'main'  
    name: Pre Validate Staging  
    uses: "./.github/workflows/template.yml"  
    permissions:  
      contents: read  
      security-events: write  
      actions: read  
    with:  
      environment: Staging  
    secrets: inherit  
  
  build-and-deploy-PROD:  
    if: contains(github.head_ref, 'hotfix') && github.base_ref == 'main'  
    name: Validate to Prod  
    uses: "./.github/workflows/template.yml"  
    permissions:  
      contents: read  
      security-events: write  
      actions: read  
    with:  
      environment: Prod  
    secrets: inherit