# securitytoken-inject

> **Security action — include in every workflow.**
> This action comes from the SOC and is highly advised to be added to all GitHub Action workflows across the organization to improve alerting against supply-chain attacks.

---

## Table of contents

- [Why this action exists](#why-this-action-exists)
- [What gets caught?](#what-gets-caught)
- [How does it work?](#how-does-it-work)
- [Inputs](#inputs)
- [Quick usage](#quick-usage)
    - [Choose the right token expiry](#choose-the-right-token-expiry)
- [Multi-job workflows](#multi-job-workflows)
- [Add additional context (optional)](#add-additional-context-optional)
- [Summary](#summary)
- [Secret provisioning](#secret-provisioning)

## Why this action exists

Supply-chain attacks via compromised CI/CD dependencies are one of the most effective vectors for credential theft. When a package or tool running inside a GitHub Actions runner is compromised - at build time, during a scan, or as part of a third-party action - attackers often attempt to harvest credentials from the runner's environment variables or well-known file locations.

This action deploys a **digital-tripwire** directly into that attack path. It injects decoy credentials into the runner environment and writes them to disk. The moment any tool tries to *validate* the credentials the SOC will receive a high-fidelity alert.

### What gets caught?

Any compromised dependency that implements credential-harvesting and automated credential-validation (e.g. via [TruffleHog](https://github.com/trufflesecurity/trufflehog)) behavior will trigger an alert. This would have caught past supply-chain attacks such as:

| Example                                                                                                                      | Attack vector                                                                                                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [**Trivy**](https://www.crowdstrike.com/en-us/blog/from-scanner-to-stealer-inside-the-trivy-action-supply-chain-compromise/) | On March 19, 2026 the popular container/filesystem vulnerability scanner Trivy was compromised, namely the package `aquasecurity/trivy-action@0.24.0` which spread credential stealer malware |
| [**PyTorch**](https://www.bleepingcomputer.com/news/security/backdoored-pytorch-lightning-package-drops-credential-stealer/) | On April 30, 2026 the popular ML python package PyTorch v.2.6.3 was compromised by the "Shai Hulud" worm which uses harvested credentials to infect other code repositories                   |
| [**GlassWorm malware**](https://www.aikido.dev/blog/glassworm-returns-unicode-attack-github-npm-vscode)                      | In March, 2026 the GlassWorm malware infected and spread to over 433 compromised components (inc. GitHub Python and JS repos, VSCode extensions and npm packages)                             |

### How does it work?

1. **Seed:** At the start of the workflow, this action calls the security endpoint API and receives a freshly minted pair of cloud credentials that are tracked server-side based on supplied information about the repository and runner id.
2. **Expose:** The credentials are written into `$GITHUB_ENV` (making them visible to all subsequent steps as environment variables) and to disk.
3. **Detect:** If any process in the runner - legitimate or malicious - attempts to use these credentials, the security solution alerts the SOC.
4. **Attribute:** The token memo is automatically populated with detailed run context - including the run ID, actor and triggering actor, repository, ref, commit SHA, workflow name/file, job, trigger event, and a direct link to the run. This allows any triggered alert to be attributable.

> **Note 1**: This security measure will not prevent a potential repo compromise through an infected resource, but it will drastically <ins>increase the SOC response time to identify and react to an incident</ins>.
> 
> **Note 2**: Even if this action somehow fails, it will not affect the rest of your workflow or cause it to crash.

---

## Inputs

| Parameter | Required | Default                                                               | Description                                                                                                                                                                                                                                                                                    |
|---|---|-----------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `auth_token` | **Yes** | must be set to `secrets.SECURITYTOKEN_APIKEY` (organizational secret) | API authentication token to dynamically generate securitytokens. Supply this from the organization-level GitHub secret provided to your workflow — never hardcode it.                                                                                                                          |
| `expiry` | No | `14d`                                                                 | Lifetime of the created token as accepted by the API. Select the expiry according to how frequently the workflow runs. Recommended values, workflows runs:<br><br>• **Multiple times per day:** `1d`<br>• **Multiple times per week:** `7d`<br>• **Multiple times per month:** `14d` (default) |
| `memo_extra` | No | *(empty)*                                                             | Free-text string appended to the auto-generated memo. Use it to add context information environment, team, or service identifiers (e.g. `env=prod,team=platform`).                                                                                                                             |
| `write_to_disk` | No | `false` (set to `true` for non-persistent, ephemeral workers)         | When `true`, the securitytoken is written to disk. This exposes it to file-system scrapers in addition to environment-variable harvesters. Set to `true` for non-persistent, ephemeral runners - which are most.                                                                                    |
| `aws_profile` | No | `snowcrash`                                                             | Named profile written into the credentials file. Deliberately set to a non-`default` value so the securitytoken does not interfere with any real credentials.                                                                                                                                  |

---

> It is likely AWS is not used within your organization - worry not. Attackers, especially their automated credential validation tools, don't know and they simply don't care.

## Quick usage

Add the action as the **first step** of your job, before any tooling that could be compromised runs:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # Injects a securitytoken into the worker env; reference the README of the action for details
      - name: Securitytoken inject
        uses: <WHERE YOU MAINTAIN YOUR ACTIONS>/securitytoken_inject@<branch or commit hash>
        continue-on-error: true
        with:
          auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }}
		  write_to_disk: false # only for self-hosted, persistent runners. Set to `true` otherwise																				  
          expiry: (1d|7d|14d) # adapt based on table below

      # ... rest of your workflow steps
```
> **Dependencies:** This action requires bash, **curl**, **jq**, grep, awk, mkdir and chmod to be available on the runner/container. See README#requirements."																																				   
### Choose the right token expiry

The `expiry` value should be adjusted based on how often the workflow generally runs to stay under your token-creation limit. Check with Thinkst Canary what that limit is.

| Workflow frequency | Recommended expiry |
|-------------------|--------------------|
| Multiple times per day | `expiry: 1d`       |
| Multiple times per week | `expiry: 7d`       |
| Multiple times per month | `expiry: 14d`      |

For workflows with multiple jobs, keep in mind that each job executing this action generates its own token. Deviate from these recommendations based on your token-limit.

## Multi-job workflows

> **Important:** Jobs in a multi-job workflow do not share environment variables, process memory, or files created by other jobs.

To protect a specific job, this action must run within that job before any third-party action, package installation, scanner, or other potentially untrusted code is executed.

If a job does not execute any third-party actions, external tooling or package installations, it is not mandatory for this action to be added.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      # Injects a securitytoken into the worker env; reference the README of the action for details
      - name: Securitytoken inject
        uses: <WHERE YOU MAINTAIN YOUR ACTIONS>/securitytoken_inject@<branch or commit hash>
        continue-on-error: true
        with:
          auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }}
          expiry: 1d
      - uses: <compromised action>

  build:
    runs-on: ubuntu-latest
    steps:
      # Injects a securitytoken into the worker env; reference the README of the action for details
      - name: Securitytoken inject
        uses: <WHERE YOU MAINTAIN YOUR ACTIONS>/securitytoken_inject@<branch or commit hash>
        continue-on-error: true
        with:
          auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }}
          expiry: 1d
      - uses: <another compromised action>
```

## Add additional context (optional)


Use the `memo_extra` parameter to tag your team and the environment or other valuable information. This makes the SOC alerts even more actionable by adding more context.

> **Hint:** This is optional. The most important information is by default already linked to your tokens and visible in your Cloud Console.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      # Injects a securitytoken into the worker env; reference the README of the action for details
      - name: Securitytoken inject
        uses: <WHERE YOU MAINTAIN YOUR ACTIONS>/securitytoken_inject@<branch or commit hash>
        continue-on-error: true
        with:
          auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }}
          memo_extra: "env=production,team=platform,service=api-gateway"
		  
      # ... rest of your workflow steps				  									  
```

## Requirements

This action requires **bash**, **curl**, **jq**, **grep**, **awk**, **mkdir** and **chmod** to be installed on the respective runner/container.
Particularly `curl` and `jq` often need to be supplied in container environments for the action to work!		   
---

## Summary

- **Always place this action before** any other actions, package installs (`npm ci`, `pip install`, `cargo build`), or security scanning tools.
- In case of **multi-job** workflows, it's highly recommended that this actions runs as first step in each job - **especially if 3rd party actions or packages are used**.
- Your workflow will still run even if this actions somehow fails. Thus, critical workflows should not be exempted from being secured with this action.
- The only mandatory parameter to supply is the  **organization-level secret** (`auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }}`) that all repositories should be able to access.
- Set `write_to_disk` to `true` for ephemeral, non-persistent workers. Disabling the feature is only a security measure for self-hosted, persistent workers.																	   
- Provide the `expiry` value according to how often your workflow runs:
    - Multiple times per day → `1d`
    - Multiple times per week → `7d`
    - Multiple times per month → `14d` (this is the default)
- Make sure the needed runtime dependencies are present, mainly `curl` and `jq`.
---

## Secret privisioning

The canarytoken API key `auth_token` must be pre-provisioned as an organization-level GitHub **Action secret** as well as **Dependabot secret** and made available to all relevant repositories. The latter is needed because there are some workflows that get run by Dependabot (e.g. triggered by a PR) which does not have access to the same organizational GitHub Action secrets. Thus, they need to be privisioned separately.