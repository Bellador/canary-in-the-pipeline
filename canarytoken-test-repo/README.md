
# Testing Canarytokens to secure CD/CI Pipelines in GitHub

## Description

This repository is used to test the concept of securing CD/CI pipelines against supply-chain attacks by injecting canarytokens into the process memory of a worker and writing it to disk.
The workflow `canarytoken-inject-test.yml` uses and tests the functionality of the GitHub action `securitytoken_inject` which contains the logic to:

- Dynamically create a canarytoken via API request.
- Logically link it to the repository and the workflow to allow for attribution and IR.
- Inject it into the worker's process memory and write it to disk before any other action.
- Alert the SOC of supply-chain attacks via compromised packages. That is, when credential stealer malware exfiltrates and validates the decoy credentials.

## Implementation

Running the test workflow will invoke the subsequent GitHub action: 
```yaml
steps:
   - name: Securitytoken inject
     uses: <WHERE YOU MAINTAIN YOUR ACTIONS>/securitytoken_inject@<branch or commit hash>
     continue-on-error: true
     with:
        auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }} # Reference the canarytoken API key that is stored as GitHub organizational secret
```

## Test results 

The test will validate if the AWS canarytoken was injected into the environment variables and was written to disk. Additionally, the workflow will measure the time it took for the action to complete. This to allow an estimated accumulated "cost" for running the action across all repositories and workflows.

![workflow-test-results](../workflow-test-result.png)