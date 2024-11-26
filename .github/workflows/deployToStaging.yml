name: Deploy to Staging Sandbox
run-name: ${{ github.actor }} is running the Github Actions 🚀

on:
  pull_request:
    types: [closed] #closed --> closed/merged
    branches:
      - develop
    paths:
      - 'force-app/**'

jobs:
  build-and-deploy: 
    name: Deploy to Staging Sandbox
    if: github.head_ref != 'main'
    uses: "./.github/workflows/template.yml"
    permissions:
      contents: read
      security-events: write
      actions: read
    with:
      environment: Staging
    secrets: inherit