
# Test Canarytokens to secure CD/CI Pipelines

## Description

This repository is used to test the concept of securing CD/CI workflows against supply-chain attacks by injecting canarytokens into the process memory of a GitHub worker.
A repository workflow is created which uses the complement action named `securitytoken_inject` which contains the logic to:

- Dynamically create a canarytoken via API request
- Logically link it to the repository and the workflow to allow for attribution
- Inject it into the worker's process memory and write it to disk before any other job step executes
- Alert the SOC of supply-chain attacks via compromised packages. That is, when credential stealer malware exfiltrates and validates the decoy credentials

## Implementation

Running the test workflow will invoke the complement action `securitytoken_inject`. Successful execution will output the AWS canarytoken in the environment variables as well as the files written to disk.
Additionally, the workflow will measure the time taken for the action to allow an estimated accumulated "cost" for running the action across all repositories and workflows.

```commandline
steps:
   - name: inject security token
     # referencing the test branch via sha pinning; for production use @develop instead!
     uses: <WHERE YOU MAINTAIN YOUR ACTION>/securitytoken_inject@main 
     with:
        auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }} # Reference the canarytoken API key that is stored as GitHub organizational secret
```