name: Pre Validate to PROD
run-name: ${{ github.actor }} is running the Github Actions 🚀

on:
  pull_request:
    types: [opened, edited, reopened, synchronize]
    branches:
      - release/*
    paths:
      - 'force-app/**'

jobs:
  build-and-deploy: 
    name: Validate to Prod
    uses: "./.github/workflows/template.yml"
    permissions:
      contents: read
      security-events: write
      actions: read
    with:
      environment: Prod
    secrets: inherit