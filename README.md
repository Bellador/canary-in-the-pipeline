# 🐤 GitHub CI/CD Canarytoken Setup

Catch supply-chain attacks in your GitHub Workflows before they turn into breaches.

## How it works

Decoy credentials (canarytokens) are injected into a GitHub runner's environment and on disk. Compromised packages or 3rd party dependencies are a direct supply-chain risk and often ship credential stealer malware. If compromised the malware harvests and auto-validates creds using tools like [TruffleHog](https://github.com/trufflesecurity/trufflehog) - that validation attempt trips the canarytoken and alerts your SOC with high-fidelity.

> **Built for [Thinkst Canary](https://canary.tools/)** AWS canarytokens, but adaptable to any supplier with a mass-deployment API.

> ⚠️ **Alerts fire only on canarytoken validation**, not on compromise alone. This targets credential-stealer malware that auto-validates harvested creds. A worker breach by itself won't trigger an alert.

## Components

| Folder | Purpose |
|---|---|
| [`securitytoken_inject`](./securitytoken_inject/README.md) | Composite action — generates & injects canarytokens |
| [`canarytoken-test-repo`](./canarytoken-test-repo/README.md) | Test workflow — verifies the setup works |

## Setup

1. **Configure the action** in `securitytoken_inject/action.yml` (search for `# CHANGE HERE!`):
   - Set your canarytoken supplier's API domain (near the top of the file)
   - Adapt the e-mail address that should be displayed to the one of your SOC or maintainer
   - Adapt the API call logic if not using Thinkst Canary

2. **Deploy** `securitytoken_inject` to your org's shared Actions/Reusable Workflows location.

3. **Get an API key** from your provider ([Thinkst Canary guide](https://help.canary.tools/hc/en-gb/articles/7111549805213-Flock-API-Keys)), store it as the org-level secret `SECURITYTOKEN_APIKEY`, and expose it to relevant repos.

4. **Configure the test workflow** in `canarytoken-test-repo/.github/workflows/canarytoken-inject-test.yml` by pointing `uses:` at your deployed action (repo + branch).

5. **Deploy** `canarytoken-test-repo`, confirm the org secret is exposed to it.

6. **Run** the *Canarytoken Effectiveness Check* workflow and review the results.

7. **Roll out** to production repos — see [deployment examples](./securitytoken_inject/README.md#quick-usage).

![Test result](./workflow-test-result.png)

## Incident response

How does your IR team respond to such an alert and attribute it to a distinct workflow run? 
During the dynamic generation of a canarytoken, the token is supplied and registered in your cloud console with data to pinpoint it's whereabouts in your GitHub environment. 
Most valuable is here the `run_url` value. It allows your IR team to step right into the exact workflow run that triggered the alert.

> **Important:** Your IR team must identify the compromised dependenc(y|ies) and rotates all secrets the worker had access to. The alert is only your detection, but not your prevention measure!

## Add it to all your workflows

The technical implementation is simple enough, but how do you get the action added to all workflows in your organization? Convincing your Devs to adopt this action and add it to their workflows can be hard solely under the premise of security.
Personally, I created a blog post in our internal software-development SharePoint to spread the message. Additionally, I posted it to the "GitHub @ Development" MS Teams channel. The post pretty much resembled the information contained in the README's of this repo - answering the questions of:
(1) What is the issue? - Supply-chain risk, (2) What do we do about it? - Increase monitoring, (3) How does it work? - Dev's want to understand the technical underpinnings, and lastly (4) How can they help? - Adding this action proactively.

Initially, I used simple searches to identify high-exposure workflows and opened issues with the call to action to add this action. Example queries could be: 

```yaml
org:example-org path:.github/workflows (path:*.yml OR path:*.yaml) "curl" "| " NOT is:archived
org:example-org path:.github/workflows (path:*.yml OR path:*.yaml) "wget" "| " NOT is:archived
org:example-org path:.github/workflows (path:*.yml OR path:*.yaml) "pip install" NOT is:archived
org:example-org path:.github/workflows (path:*.yml OR path:*.yaml) "npm install -g" NOT is:archived
```

Lastly, think about automating the previous step. Create automated issues for repositories that match certain exposure-searches and force the Dev's to review it and add the action.
If the issue is not treated properly or is forgotten, the repo should risk getting archived. 

> Important: Don't forget to implement preventive security measures to protect your CI/CD pipelines next to this action alone, such as sha-pinning of GitHub Actions and [more](https://docs.github.com/en/actions/reference/security/secure-use).

## Inspiration

- [https://tracebit.com/blog/detecting-cicd-supply-chain-attacks-with-canary-credentials](https://tracebit.com/blog/detecting-cicd-supply-chain-attacks-with-canary-credentials)
- [https://grafana.com/blog/canary-tokens-learn-all-about-the-unsung-heroes-of-security-at-grafana-labs/](https://grafana.com/blog/canary-tokens-learn-all-about-the-unsung-heroes-of-security-at-grafana-labs/)
