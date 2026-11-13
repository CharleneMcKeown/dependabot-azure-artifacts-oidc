# Dependabot → Azure Artifacts with OIDC on GHE.com

How to let **real Dependabot jobs** authenticate to a **private Azure Artifacts NuGet feed** using **one** Entra managed identity and **one** standard federated identity credential (FIC): no PATs, no client secrets and no per-repository identities. Written for GitHub Enterprise Cloud with data residency (GHE.com).

> This is field-tested guidance, not official GitHub documentation, and product behavior may change. It was validated in November 2026 for the `nuget-feed` registry type only; see [What was not tested](#what-was-not-tested).

## Why a custom subject is needed

Dependabot's OIDC token uses the same issuer and subject template as GitHub Actions. By default the subject is repository-specific. On repositories with immutable subjects enabled (every repository we tested) it embeds the numeric organization and repository IDs:

```text
repo:<ORG>@<ORG_ID>/<REPO>@<REPO_ID>:ref:refs/heads/main
```

A standard FIC matches its subject exactly, so out of the box each repository would need its own credential. Customizing the subject template to one owner-level value lets every opted-in repository share a single FIC:

```text
repository_owner_id:<OWNER_ID>
```

## What was validated

Test setup: private, organization-owned repositories on a GHE.com tenant (all with `use_immutable_subject: true`), a private Azure Artifacts NuGet feed with no upstream sources, and one user-assigned managed identity with a single standard FIC for `repository_owner_id:<OWNER_ID>`. Dependabot version updates ran as `Dependabot Updates` workflow runs.

| Scenario | Result |
| --- | --- |
| Default subject | **Fails.** Entra rejects the repository-specific subject with `AADSTS700213`, the feed then answers HTTP 401 / `TF400813`, and Dependabot reports that it can't authenticate to a private package registry. |
| Repository-level template (`use_default: false`, `include_claim_keys: ["repository_owner_id"]`) | **Works.** Dependabot jobs in two repositories succeeded and opened update PRs. |
| Organization-level template, repository not opted in | **Fails like the default.** The org template has no effect while a repository has `use_default: true`. |
| Organization-level template + repository opt-in (`{"use_default": false}` only) | **Works.** The repository inherits the org template. After opting in, a run on the exact same commit that had just failed succeeded and opened an update PR. |

Manual Actions runs that printed only an allowlist of claims showed `sub` as `repo:<ORG>@<ORG_ID>/<REPO>@<REPO_ID>:ref:refs/heads/main` before opt-in and `repository_owner_id:<OWNER_ID>` after.

**Limits of the evidence:** Dependabot JWTs were never decoded. For the default case the subject comes from Entra's error message. For opted-in repositories the Dependabot subject is inferred from successful authentication against a FIC that only matches `repository_owner_id:<OWNER_ID>`.

## Setup

Placeholders: `<SUBDOMAIN>` is your GHE.com subdomain, `<ORG>` is the organization that owns the repositories, and `<OWNER_ID>` is its numeric ID. `gh` must be signed in to the GHE.com host (`gh auth login --hostname <SUBDOMAIN>.ghe.com`). Look up the ID with:

```bash
gh api --hostname <SUBDOMAIN>.ghe.com orgs/<ORG> --jq .id
```

### 1. Customize the OIDC subject template

**Option A: per repository**

```bash
echo '{"use_default":false,"include_claim_keys":["repository_owner_id"]}' |
  gh api --hostname <SUBDOMAIN>.ghe.com --method PUT \
    repos/<ORG>/<REPO>/actions/oidc/customization/sub --input -
```

**Option B: organization template, repositories opt in**

```bash
# Once per organization
echo '{"include_claim_keys":["repository_owner_id"]}' |
  gh api --hostname <SUBDOMAIN>.ghe.com --method PUT \
    orgs/<ORG>/actions/oidc/customization/sub --input -

# For each repository that should use the template
echo '{"use_default":false}' |
  gh api --hostname <SUBDOMAIN>.ghe.com --method PUT \
    repos/<ORG>/<REPO>/actions/oidc/customization/sub --input -
```

The organization call needs an org-level token scope (`write:org` for classic personal access tokens and OAuth tokens, per the [REST docs](https://docs.github.com/en/rest/actions/oidc)); a token without it gets `403`. Repositories that don't opt in keep the default subject.

Confirm a repository's settings:

```bash
gh api --hostname <SUBDOMAIN>.ghe.com repos/<ORG>/<REPO>/actions/oidc/customization/sub
```

Expect `use_default: false`, `include_claim_keys: ["repository_owner_id"]` and `use_immutable_subject: true`.

### 2. Create one federated credential

```bash
az identity federated-credential create \
  --name <FIC_NAME> \
  --identity-name <MANAGED_IDENTITY_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --issuer https://token.actions.<SUBDOMAIN>.ghe.com \
  --subject repository_owner_id:<OWNER_ID> \
  --audiences api://AzureADTokenExchange
```

We used a standard (not flexible) FIC on a user-assigned managed identity. This one credential serves every opted-in repository in the owner.

### 3. Give the identity access to the feed

As tested, the identity needed a Basic access level in the Azure DevOps organization, Reader on the project and Reader on the feed. No Azure RBAC role assignments were needed. See [Use service principals and managed identities in Azure DevOps](https://learn.microsoft.com/azure/devops/integrate/get-started/authentication/service-principal-managed-identity).

### 4. Configure Dependabot and NuGet

1. Copy [`examples/dependabot.yml`](examples/dependabot.yml) to `.github/dependabot.yml` and [`examples/NuGet.config`](examples/NuGet.config) to `NuGet.config` in the repository root.
2. Replace `<ADO_ORG>`, `<PROJECT>` and `<FEED>` in both feed URLs. Set `<TENANT_ID>` and `<CLIENT_ID>` to the Entra tenant and the managed identity's client ID. No PAT, password or client secret is involved.
3. The sample `NuGet.config` uses `<clear />` so only the private feed is used. If the repository already has a NuGet configuration, merge the private source in and keep any other sources the build needs.

After changing OIDC settings, re-run Dependabot with **Check for updates** (Insights → Dependency graph → Dependabot → Recent update jobs) instead of waiting for the schedule. See [Re-running Dependabot jobs](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/manage-your-dependency-security/re-run-dependabot-jobs).

## Security considerations

- **No repository or branch isolation.** Every opted-in repository presents the same subject, so any job there that can request an OIDC token for the `api://AzureADTokenExchange` audience (`permissions: id-token: write`) can use the identity. Use a dedicated identity with feed-read-only access and no Azure RBAC, and opt in only the repositories that need it.
- **Actions subjects change too.** The template also applies to ordinary Actions jobs in each opted-in repository. Inventory existing Azure federations that match `repo:...` subjects before rollout, because they will stop matching.
- Only `repository_owner_id` was tested. Adding claims changes the subject, and the FIC must then match it exactly.

## Troubleshooting

- **Dependabot says it can't authenticate to a private package registry, or the feed returns 401 / `TF400813`.** Open the failed `Dependabot Updates` run and search its log for `AADSTS700213`. The message names the subject Entra received:

  ```text
  AADSTS700213: No matching federated identity record found for presented assertion subject 'repo:<ORG>@<ORG_ID>/<REPO>@<REPO_ID>:ref:refs/heads/main'
  ```

  A repository-specific subject like this means the repository isn't using the template. Otherwise compare the subject with the FIC's issuer, subject and audience. The message is on one long line, so `grep -oE "AADSTS700213[^']*'[^']*'"` extracts it from a downloaded log.
- **The org template had no effect.** Repositories with `use_default: true` ignore it. Opt each repository in with `{"use_default": false}`.
- **`sub_claim_prefix` still looks repository-specific.** In our tests it kept a repository-specific value after the template was applied, so don't use it to judge the effective subject. Check `use_default` and `include_claim_keys`, then the Dependabot result.
- **The org-level `use_immutable_subject` reads `false`.** In our tests the org-level value read `false` while each repository reported `true`. Check the repository-level value.

## What was not tested

- Registry types and ecosystems other than `nuget-feed` / `nuget`. The [Dependabot options reference](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference) lists `tenant-id` and `client-id` OIDC authentication for several other registry types.
- Dependabot security updates (only version updates ran).
- Flexible FICs, app registrations (a user-assigned managed identity was used) and multi-tenant setups.
- Repositories using the legacy name-based subject (`use_immutable_subject: false`).
- Claims other than `repository_owner_id`.

## References

- [OpenID Connect reference: customizing the token claims](https://docs.github.com/en/actions/reference/security/oidc#customizing-the-token-claims)
- [REST API: OIDC subject claim customization](https://docs.github.com/en/rest/actions/oidc)
- [Dependabot options reference](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference)
- [Re-running Dependabot jobs](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/manage-your-dependency-security/re-run-dependabot-jobs)
- [Workload identity federation (Microsoft Entra)](https://learn.microsoft.com/entra/workload-id/workload-identity-federation)
- [`az identity federated-credential`](https://learn.microsoft.com/cli/azure/identity/federated-credential)
- [Service principals and managed identities in Azure DevOps](https://learn.microsoft.com/azure/devops/integrate/get-started/authentication/service-principal-managed-identity)
