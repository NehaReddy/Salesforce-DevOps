name: Manually Pre Validate to Staging Sandbox
run-name: ${{ github.actor }} is running the Github Actions 🚀

on:
  workflow_dispatch:
    branches:
      - release/*

jobs:
  build-and-deploy: 
    name: Manually Pre Validate to Staging Sandbox
    uses: "./.github/workflows/template.yml"
    permissions:
      contents: read
      security-events: write
      actions: read
    with:
      environment: Staging
    secrets: inherit