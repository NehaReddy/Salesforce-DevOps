name: Pre Validate to Staging Sandbox
run-name: ${{ github.actor }} is running the Github Actions 🚀

on:
  pull_request:
    types: [opened, edited, reopened, synchronize]
    branches:
      - develop
    paths:
      - 'force-app/**'

jobs:
  build-and-deploy:
    name: Pre Validate Staging
    if: github.head_ref != 'main'
    uses: "./.github/workflows/template.yml"
    permissions:
      contents: read
      security-events: write
      actions: read
    with:
      environment: Staging
    secrets: inherit