# setup-ezgh

A GitHub Action that installs the [EZGH Cloud](https://ezghcloud.com) CLI, `ezgh`, and lets the job
act as one of your organization's **bots without storing an API key**. GitHub gives the job an
OpenID Connect token; EZGH Cloud checks it against the bot's OIDC trust and returns an API key that
acts as the bot and expires on its own (after 1 hour by default).

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write   # lets the job get an OIDC token from GitHub
    steps:
      - uses: actions/checkout@v7
      - uses: ezgamehost/setup-ezgh@v1
        with:
          org: org_k3f9a0x2m7qp
          bot: bot_8k2l4m6n8p0q
      - run: ezgh projects list
```

Later steps get `EZGH_API_KEY` (masked in logs), `EZGH_ORG` and `EZGH_DOMAIN`, so `ezgh`, and
anything else that reads them, acts as the bot.

Without `org` and `bot` the action only installs `ezgh`, for jobs that authenticate some other way.

## Before the first run

Do this once, with an account that can manage IAM, in the console (**IAM → OIDC providers**, then
the bot's **OIDC trust** tab) or with `ezgh`:

1. **Register GitHub as an OIDC provider** for your organization:

   ```sh
   ezgh iam oidc-providers create github \
     --issuer https://token.actions.githubusercontent.com
   ```

   Its audience defaults to your domain (`ezghcloud.com`), which is what this action asks GitHub
   for; set `--audiences` and the `audience` input together to use another.

2. **Say which jobs may act as the bot**, with a trust entry matching the token's `sub`:

   ```sh
   # Only the main branch of acme/app:
   ezgh iam bots oidc-trust add deployer --provider github \
     --subject 'repo:acme/app:ref:refs/heads/main'

   # Only jobs that use the production environment:
   ezgh iam bots oidc-trust add deployer --provider github \
     --subject 'repo:acme/app:environment:production'
   ```

   `*` matches any run of characters (`repo:acme/app:*` is every branch, tag and pull request of
   the repository). A GitHub subject must name the owner (`repo:acme/…`), unless the entry pins
   `repository_owner_id` or `repository_id` with `--claim`, which also survive renames. Extra
   `--claim name=value` conditions, such as
   `job_workflow_ref=acme/app/.github/workflows/deploy.yml@refs/heads/main`, must equal the token's
   claims exactly.

3. **Give the bot only what the job needs**, with policies, as for any bot. The key has exactly the
   bot's access.

## Inputs

| Input | Default | |
| --- | --- | --- |
| `org` | | The organization, by ID or slug (`org_…`). With `bot`, exchanges the job's OIDC token. |
| `bot` | | The bot to act as, by ID or slug (`bot_…`). With `org`, exchanges the job's OIDC token. |
| `version` | latest | The `ezgh` version to install, such as `0.7.0`. |
| `domain` | `ezghcloud.com` | The EZGH Cloud domain. |
| `audience` | the domain | The audience to ask GitHub for: one of the OIDC provider's audiences. |
| `expires-in` | `1h` | How long the key is valid, `15m` to `12h`. |

Names can't be looked up without a credential, so `org` and `bot` are IDs or slugs, not names.

## Security

- **Pin the action** to a release tag, or to a commit SHA for the strictest supply-chain policy.
- **Keep trust entries narrow.** Prefer a branch or an environment over `repo:acme/app:*`: pull
  requests get tokens too (never from forks, which can't get an OIDC token at all). GitHub
  environments with required reviewers make a good boundary for production bots.
- `ezgh` is installed from `get.ezghcloud.com`, and its archive is checked against the release's
  `checksums.txt` before it's used.
- The key never touches disk outside the runner's environment file, is masked in logs, and
  expires on its own. Removing the trust entry or the provider ends it at once. Each exchange is
  recorded in your organization's Trails as `CreateOidcApiKey`, and everything the key does
  names the job's subject.

## Troubleshooting

- **"This job can't get an OIDC token"**: add `permissions: id-token: write` to the job.
- **"The OIDC token wasn't accepted" (401)**: every refusal looks the same, so that nobody can
  probe your organization. `ezgh` prints the token's `iss`, `aud`, `sub` and common claims; compare
  them with the provider's issuer and audiences and the bot's trust entries.
- **"can't exchange OIDC tokens"**: the installed `ezgh` is too old; set `version` to a newer one.

Other CI systems (GitLab CI, Buildkite, Kubernetes service accounts…) use the same exchange
directly. On GitLab CI, for example:

```yaml
deploy:
  id_tokens:
    EZGH_ID_TOKEN:
      aud: ezghcloud.com
  script:
    - export EZGH_API_KEY=$(ezgh auth oidc-exchange --org org_k3f9a0x2m7qp --bot bot_8k2l4m6n8p0q --token-env EZGH_ID_TOKEN)
    - ezgh projects list
```

See the [EZGH Cloud docs](https://docs.ezghcloud.com).

## License

[MIT](LICENSE)
