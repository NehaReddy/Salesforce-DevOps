name: Resuable workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      ENCRYPTED_KEY_FILE:
        required: true
      JWT_KEY_FILE:
        required: true
      KEY:
        required: true
      IV:
        required: true
      SF_CLIENT_ID:
        required: true
      SF_INSTANCE_URL:
        required: true
      SF_USERNAME:
        required: true

jobs:

  build:
    permissions:
      contents: read # for actions/checkout to fetch code
      security-events: write # for github/codeql-action/upload-sarif to upload SARIF results
      actions: read # only required for a private repository by github/codeql-action/upload-sarif to get the Action run status

    runs-on: { group: ubuntu-uhg } ## Github Hosted [UHG Hosted]
    environment: ${{ inputs.environment }}

    ## Create the variables for the Job and store the PR body in the variables
    env:
      PR_BODY: ${{ github.event.pull_request.body }}
      NULL_VAR: ''
      TEST_LEVEL: RunSpecifiedTests
    steps:
      - name: "Print the Information"
        run: |
          echo ${{ github.event }}
          echo ${{ github.event.action }}
          echo ${{ github.ref }}
          echo ${{ env.TEST_LEVEL }}
        
      - name: Checkout Code
        uses: actions/checkout@v4.1.7
        with:
          fetch-depth: 0
      - name: npm install
        run: echo "running npm install"

      - name: Run the extractor script
        id: extract_classes
        run: |
          echo "${{ github.event.pull_request.body }}"  >> pr_body.txt
          python PRBODY_TESTCLASS.py > output.txt
          cat output.txt
          echo "apex_classes=$(cat output.txt)" >> $GITHUB_ENV
          echo "${{ env.apex_classes }}"
        
        ## Install Saleforce CLI
      - name: Install Salesforce CLI
        run: npm install @salesforce/cli --global 

      - name: Install Salesforce Code Anayzer
        id: salesforce-cli-scanner
        run: sf plugins install @salesforce/sfdx-scanner

      - name: Run Salesforce Code Anayzer Scan
        id: code-analyzer
        run: |
          mkdir reports
          echo "Folder is created"
          
          sf scanner run --format html --target force-app/main/default/classes --engine pmd,pmd-appexchange --category Design,Best Practices, Code Style,Performance,Security,Documentation, Error Prone --outfile reports/scan-reports.html  
          # echo "Starting the scan in sarif format"
          
          # sf scanner run --format sarif --target force-app/main/default/classes --engine pmd,pmd-appexchange --category Design,Best Practices, Code Style,Performance,Security,Documentation, Error Prone --outfile reports/scan-reports.sarif --severity-threshold=3 
          # sf scanner run --format sarif --target force-app/main/default/classes --engine pmd,pmd-appexchange --category Design,Best Practices, Code Style,Performance,Security,Documentation, Error Prone --outfile reports/scan-reports.sarif
          # echo "Scanning is Completed"
      
      ## Upload the report results as artifacts
      - name: Upload a Salesforce CLI Scan Report
        id: upload-reports

        if: always()
        uses: actions/upload-artifact@v4.4.0
        with:
          name: cli-scan-report
          path: reports/scan-reports.html
          
      - name: Decrypt the server.key.enc file
        run: openssl enc -nosalt -aes-256-cbc -d -in ${{ secrets.ENCRYPTED_KEY_FILE }} -out ${{ secrets.JWT_KEY_FILE }} -base64 -K ${{ secrets.KEY }} -iv ${{ secrets.IV }}
      
      - name: Authorize with Salesforce org
        run: |
          echo ${{vars.ORG_URL}}

          sf org login jwt --username ${{ secrets.SF_USERNAME }} --jwt-key-file ${{ secrets.JWT_KEY_FILE }} --client-id ${{ secrets.SF_CLIENT_ID }} --set-default --alias ci-org --instance-url ${{ secrets.SF_INSTANCE_URL }}

      

      - name: Check if apex_classes has a value
        run: |
          if [[ -n "${{ env.apex_classes }}" ]]; then
            echo "apex_classes has a value: '${{ env.apex_classes }}'. Executing code..."
            # Place your code here that should run if apex_classes is not empty
          else
            echo "apex_classes is empty."
          fi
      ## Check the code coverage -- END
      
      - name: Set test level
        run: |
          if [ "${{ env.apex_classes }}" = "NONE" ]; then
            echo "TEST=-l NoTestRun" >> $GITHUB_ENV
          elif [ "${{ env.apex_classes }}" = "ALL" ]; then
            echo "TEST=-l RunLocalTests" >> $GITHUB_ENV
          else
            echo "TEST=-l RunSpecifiedTests -r ${{ env.apex_classes }}" >> $GITHUB_ENV
          fi         
      - name: Install SFDX Git delta aka SGD 
        run: |
          echo 'y' | sf plugins install sfdx-git-delta
      
      - name: Create delta folder 
        run: |
          mkdir delta
      
      - name: Generate the delta file for deployment
        run: |
          sf sgd source delta --to "HEAD" --from "HEAD~1" --output "./delta" --generate-delta --ignore-whitespace --ignore .sgdignore
          echo "--- package.xml generated with added and modified metadata ---"
          cat delta/package/package.xml
          cat delta/destructiveChanges/destructiveChanges.xml

      - name: Validate The Code to Salesforce
        if: ${{ github.event_name == 'pull_request' && github.event.action != 'closed' && env.apex_classes != 'No Apex classes found' }}
        run: |
          echo ${{ github.event_name }}
          echo ${{ github.event.action }}
          echo ${{ github.event.pull_request.merged }}

          if grep -q '<types>' delta/package/package.xml ; then
            echo "---- validating added and modified metadata ----"
            sf project deploy validate --source-dir delta/force-app --target-org ci-org --test-level RunSpecifiedTests --tests ${{ env.apex_classes }} --wait 10
          else
            echo "---- No changes to validate ----"
          fi

      - name: Validate The Code to Salesforce
        if: ${{ github.event_name == 'pull_request' && github.event.action != 'closed' && env.apex_classes == 'No Apex classes found' }}
        run: |
          echo ${{ github.event_name }}
          echo ${{ github.event.action }}
          echo ${{ github.event.pull_request.merged }}

          if grep -q '<types>' delta/package/package.xml ; then
            echo "---- validating added and modified metadata ----"
            sf project deploy validate --source-dir delta/force-app --target-org ci-org --wait 10
          else
            echo "---- No changes to validate ----"
          fi

      - name: Validate Pre Destructive Changes
        id: ValidatePreDestructiveChanges
        if: ${{ github.event_name == 'pull_request' && github.event.action != 'closed' }}
        run: |
          if grep -q '<types>' delta/destructiveChanges/destructiveChanges.xml ; then
            echo "---- validating Destructive Changes ----"
            sf project deploy validate --target-org ci-org --pre-destructive-changes delta/destructiveChanges/destructiveChanges.xml --manifest delta/destructiveChanges/package.xml --wait 10
          else
            echo "---- No changes to validate ----"
          fi

      - name: Validate Post Destructive Changes
        id: ValidatePostDestructiveChanges
        if: ${{ github.event_name == 'pull_request' && github.event.action != 'closed' }}
        run: |
          if grep -q '<types>' destructiveChanges/postDestructiveChanges.xml ; then
            echo "---- deploying Destructive Changes ----"
            sf project deploy start --target-org ci-org --post-destructive-changes destructiveChanges/postDestructiveChanges.xml --manifest destructiveChanges/package.xml --wait 10
          else
            echo "---- No changes to validate ----"
          fi

      - name: Deploy Pre Destructive Changes
        if: ${{ github.event.pull_request.merged == true && github.event.action == 'closed' }}
        run: |
          if grep -q '<types>' delta/destructiveChanges/destructiveChanges.xml ; then
            echo "---- deploying added and modified metadata ----"
            sf project deploy validate --target-org ci-org --pre-destructive-changes delta/destructiveChanges/destructiveChanges.xml --manifest delta/destructiveChanges/package.xml
          else
            echo "---- No changes to validate ----"
          fi
      - name: Deploy
        if: ${{ github.event.pull_request.merged == true && github.event.action == 'closed' }}
        run: |
          echo ${{ github.event_name }}
          echo ${{ github.event.action }}
          echo ${{ github.event.pull_request.merged }}
          
          if grep -q '<types>' delta/package/package.xml ; then
            echo "---- Deploying added and modified metadata ----"
            sf project deploy start --source-dir delta/force-app --target-org ci-org
          else
            echo "---- No changes to deploy ----"
          fi
