# securitytoken-inject

> **Security action — include in every workflow.**
> This action comes from the SOC and is highly advised to be added to all GitHub Action workflows across the organization to improve alerting against supply-chain attacks.

---

## Why this action exists

Supply-chain attacks via compromised CI/CD dependencies are one of the most effective vectors for credential theft. When a package or tool running inside a GitHub Actions runner is compromised — at build time, during a scan, or as part of a third-party action — attackers often attempt to harvest credentials from the runner's environment variables or well-known file locations.

This action deploys a **honeytoken trap** directly into that attack path. It injects decoy credentials into the runner environment and writes them to disk. The moment any tool tries to *validate* the credentials the SOC will automatically receive a high-fidelity alert.

### What gets caught?

Any compromised dependency that implements credential-harvesting and automated credential-validation behavior (e.g. via [TruffleHog](https://github.com/trufflesecurity/trufflehog)) will trigger an alert. This would have caught past supply-chain attacks such as:

| Example                                                                                                                      | Attack vector                                                                                                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [**Trivy**](https://www.crowdstrike.com/en-us/blog/from-scanner-to-stealer-inside-the-trivy-action-supply-chain-compromise/) | On March 19, 2026 the popular container/filesystem vulnerability scanner Trivy was compromised, namely the package `aquasecurity/trivy-action@0.24.0` which spread credential stealer malware |
| [**PyTorch**](https://www.bleepingcomputer.com/news/security/backdoored-pytorch-lightning-package-drops-credential-stealer/) | On April 30, 2026 the popular ML python package PyTorch v.2.6.3 was compromised by the "Shai Hulud" worm which uses harvested credentials to infect other code repositories                   |
| [**GlassWorm malware**](https://www.aikido.dev/blog/glassworm-returns-unicode-attack-github-npm-vscode)                      | In March, 2026 the GlassWorm malware infected and spread to over 433 compromised components (inc. GitHub Python and JS repos, VSCode extensions and npm packages)                             |

### How does it work?

1. **Seed:** At the start of the workflow, this action calls the security endpoint API and receives a freshly minted pair of cloud credentials that are tracked server-side based on supplied information about the repository and runner id.
2. **Expose:** The credentials are written into `$GITHUB_ENV` (making them visible to all subsequent steps as environment variables) and to disk.
3. **Detect:** If any process in the runner — legitimate or malicious — attempts to use these credentials, the security solution alerts the SOC.
4. **Attribute:** The token memo is automatically populated with the GitHub run ID, actor, repository, ref, and commit SHA, so any triggered alert can be attributed back to the exact workflow run.

> **Note 1**: This security measure will not prevent a potential compromise through an injected package, but it will drastically <ins>increase the SOC's response time to identify and react to the incident</ins>.
> 
> **Note 2**: Even if this action somehow fails, it will not affect the rest of your workflow or cause it to crash

---

## Inputs

| Parameter | Required | Default                                               | Description                                                                                                                                                                                                 |
|---|---|-------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `auth_token` | **Yes** | security.securitytoken-apikey (organizational secret) | API authentication token to dynamically generate securitytokens. Supply this from the organization-level GitHub secret provided to your workflow — never hardcode it.                                       |
| `canary_domain` | No | `<CAN BE DEFINED>`                                    | Hostname of the security console that hosts the API endpoint.                                                                                                                                               |
| `expiry` | No | `14d`                                                 | Lifetime of the created token as accepted by the API (e.g. `1d`, `7d`, `14d`). For high-frequency workflows, use a shorter expiry to avoid hitting the console's token creation threshold.  |
| `memo_extra` | No | *(empty)*                                             | Free-text string appended to the auto-generated memo. Use it to add context information such as environment, team, or service identifiers (e.g. `env=prod,team=platform`).  |
| `write_to_disk` | No | `true`                                                | When `true`, the securitytoken is written to disk. This exposes it to file-system scrapers in addition to environment-variable harvesters. Disable only if your runner has no writable home directory. |
| `credentials_path` | No | `/home/runner/.aws/credentials`                       | Absolute path of the file written to disk. The default matches the standard AWS CLI location and should not be changed since it matches the securitytoken type.  |
| `aws_profile` | No | `aurora`                                              | Named profile written into the credentials file. Deliberately set to a non-`default` value so the securitytoken does not interfere with any real credentials.  |
| `fail_on_error` | No | `false`                                               | When `true`, a failure to reach the API or parse its response will fail the workflow step. Default is set to `false` to not jeopardize the overall workflow.  |

---


## Quick Usage

Add the action as the **first step** of your job, before any tooling that could be compromised runs:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Inject securitytoken
        uses: <WHERE YOU MAINTAIN YOUR ACTION>/securitytoken_inject@main
        with:
          auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }}

      # ... rest of your workflow steps
```

### With environment and team tagging

> **Hint:** Adding some meaningful context information to the `memo_extra` really helps your SOC to attribute the alert quickly :) 

Use `memo_extra` to make SOC alerts immediately actionable by identifying which team and environment a triggered canarytoken belongs to:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Inject securitytoken
        uses: <WHERE YOU MAINTAIN YOUR ACTION>/securitytoken_inject@main
        with:
          auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }}
          memo_extra: "env=production,team=platform,service=api-gateway"

      # ... rest of your workflow steps
```

### High-frequency workflows (short token expiry)

For pipelines that run many times per day, reduce the token lifetime to avoid hitting the console's token creation limit:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Inject securitytoken
        uses: <WHERE YOU MAINTAIN YOUR ACTION>/securitytoken_inject@main
        with:
          auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }}
          expiry: "1d"
          memo_extra: "env=ci,team=backend"

      # ... rest of your workflow steps
```

### Multi-job workflows

> **Important:** In case your workflow is comprised of *multiple* jobs, this security action needs to be added as the first step to **each** job!

Jobs in a multi-job workflow work in parallel but do not share the same environment variables or the same process memory. 
Thus, it's important that this composite security action is placed as the first step in each job to fulfill its purpose.   

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Inject securitytoken
        uses: <WHERE YOU MAINTAIN YOUR ACTION>/securitytoken_inject@main
        with:
          auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }}
          memo_extra: "env=ci,team=backend"
      - uses: <compromised action>

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: <WHERE YOU MAINTAIN YOUR ACTION>/securitytoken_inject@main
        with:
          auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }}
          memo_extra: "job=build"
      - uses: <another compromised action>
```

---

## Summary

- **Always place this action before** any other actions, package installs (`npm ci`, `pip install`, `cargo build`), or security scanning tools.
- In case of **multi-job** workflows, this action needs to be placed as the first step of each job.
- Your workflow will still run even if this action somehow fails. So critical workflows are not exempted from securing via this action.
- The only mandatory parameter to supply is the **organization-level secret** (`auth_token: ${{ secrets.SECURITYTOKEN_APIKEY }}`) that all repositories can access.

---

## Organizational secret: `auth_token`

The canarytoken API key `auth_token` should be pre-provisioned as an organization GitHub secret and made available to all relevant repositories.