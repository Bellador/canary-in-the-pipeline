# 🐤 GitHub CI/CD Canarytoken Setup

Catch supply-chain attacks in your GitHub Workflows before they turn into breaches.

## How it works

This repo injects decoy credentials (canarytokens) into the runner's environment and disk. Credential-stealer malware harvests and auto-validates creds using tools like [TruffleHog](https://github.com/trufflesecurity/trufflehog) — that validation attempt trips the token and alerts your SOC.

> **Built for [Thinkst Canary](https://canary.tools/)** AWS canarytokens, but adaptable to any supplier with a mass-deployment API.

> ⚠️ **Alerts fire only on validation**, not on compromise alone. This targets credential-stealer malware that auto-validates harvested creds. A worker breach by itself won't trigger an alert.

## Components

| Folder | Purpose |
|---|---|
| [`securitytoken_inject`](./securitytoken_inject/README.md) | Composite action — generates & injects canarytokens |
| [`canarytoken-test-repo`](./canarytoken-test-repo/README.md) | Test workflow — verifies the setup works |



## Setup

1. **Configure the action** in `securitytoken_inject/action.yml`:
   - Set your canarytoken supplier's API domain (near the top of the file)
   - Adapt the API call logic if not using Thinkst Canary

2. **Deploy** `securitytoken_inject` to your org's shared Actions/Reusable Workflows location.

3. **Get an API key** from your provider ([Thinkst Canary guide](https://help.canary.tools/hc/en-gb/articles/7111549805213-Flock-API-Keys)), store it as the org-level secret `SECURITYTOKEN_APIKEY`, and expose it to relevant repos.

4. **Configure the test workflow** in `canarytoken-test-repo/.github/workflows/canarytoken-inject-test.yml`:
   - Point `uses:` at your deployed action (repo + branch)
   - Make sure `CREDS_PATH` matches your action's disk-write location
   - Make sure the AWS `--profile` is the same as in your action's

5. **Deploy** `canarytoken-test-repo`, confirm the org secret is exposed to it.

6. **Run** the *Canarytoken Effectiveness Check* workflow and review results.

7. **Roll out** to production repos — see [deployment examples](./securitytoken_inject/README.md#quick-usage).

![Test result](./workflow-test-result.png)

## Inspiration

[https://tracebit.com/blog/detecting-cicd-supply-chain-attacks-with-canary-credentials](https://tracebit.com/blog/detecting-cicd-supply-chain-attacks-with-canary-credentials)
[https://grafana.com/blog/canary-tokens-learn-all-about-the-unsung-heroes-of-security-at-grafana-labs/](https://grafana.com/blog/canary-tokens-learn-all-about-the-unsung-heroes-of-security-at-grafana-labs/)
