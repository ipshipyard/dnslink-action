# DNSLink Action

This GitHub Action sets the [DNSLink](https://www.dnslink.dev/) TXT record for a given domain.

Supports the following DNS providers:

- Cloudflare
- DNSimple
- Gandi

This action is built and maintained by [Interplanetary Shipyard](http://ipshipyard.com/).
<a href="http://ipshipyard.com/"><img align="right" src="https://github.com/user-attachments/assets/39ed3504-bb71-47f6-9bf8-cb9a1698f272" /></a>

It's a [composite action](https://docs.github.com/en/actions/sharing-automations/creating-actions/about-custom-actions#composite-actions) that can be called as a step in your workflow to set the DNSLink TXT record for a given domain.

## Table of Contents

- [Inputs](#inputs)
  - [Required Inputs](#required-inputs)
  - [Optional Inputs](#optional-inputs)
- [Usage](#usage)
  - [Simple Workflow (No Fork PRs)](#simple-workflow-no-fork-prs)
  - [Dual Workflows (With Fork PRs)](#dual-workflows-with-fork-prs)
- [FAQ](#faq)

## Inputs

### Required Inputs

| Input                 | Description                                                                                                              |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `cid`                 | CID of the build to update the DNSLink for                                                                               |
| `dnslink_domain`      | Domain to update the DNSLink for e.g. if you set docs.ipfs.tech, the \_dnslink.docs.ipfs.tech TXT record will be updated |
| `cf_zone_id`          | Cloudflare Zone ID                                                                                                       |
| `cf_auth_token`       | Cloudflare API token                                                                                                     |
| `dnsimple_token`      | DNSimple API token                                                                                                       |
| `dnsimple_account_id` | DNSimple account ID                                                                                                      |
| `gandi_pat`           | Gandi Personal Authorization Token (Bearer auth)                                                                         |
| `gandi_rrset_name`    | Name of the record for which to update the DNSLink record, e.g. `mysubdomain`, `_dnslink.` will be prepended)                               |

### Optional Inputs

| Input               | Description                                                                                    |
| ------------------- | ---------------------------------------------------------------------------------------------- |
| `cf_record_id`      | Cloudflare Record ID. If omitted, the action auto-discovers or creates the record (see below) |
| `set_github_status` | Set the GitHub commit status with the DNSLink domain and CID                                   |
| `github_token`      | GitHub token                                                                                   |

### Cloudflare Setup

#### Finding your Zone ID

1. Log in to the [Cloudflare dashboard](https://dash.cloudflare.com/)
2. Select your domain
3. On the Overview page, scroll down to the **API** section in the right sidebar
4. Copy the **Zone ID** and save it as `CF_DNS_ZONE_ID` secret in your repository

See [Cloudflare docs](https://developers.cloudflare.com/fundamentals/setup/find-account-and-zone-ids/) for more details.

#### Creating an API Token

1. Go to [API Tokens](https://dash.cloudflare.com/profile/api-tokens) in your Cloudflare profile
2. Click **Create Token**
3. Select **Create Custom Token**
4. Configure the token:
   - **Token name**: e.g., `dnslink-action for example.com`
   - **Permissions**: `Zone` / `DNS` / `Edit`
   - **Zone Resources**: `Include` / `Specific zone` / select your domain
5. Click **Continue to summary**, then **Create Token**
6. Copy the token value and save it as `CF_DNS_AUTH_TOKEN` secret in your repository

See [Cloudflare video tutorial](https://developers.cloudflare.com/videos/create-api-tokens/) for a walkthrough.

### Security: Sandboxed DNSLink Domain

For enhanced security, you can isolate DNSLink records on a separate domain. This ensures that even if your API token is compromised, attackers can only modify TXT records on the sandboxed domain, not your main domain.

**Setup:**

1. Create a dedicated zone for DNSLink records (e.g., `dnslinks.example.com`)
2. Create an API token scoped only to that zone
3. On your main domain, add a CNAME pointing to the sandboxed record:
   ```
   _dnslink.docs.yourdomain.com  CNAME  _dnslink.docs.dnslinks.example.com
   ```
4. Set `dnslink_domain` to the sandboxed domain (not your main domain):
   ```yaml
   dnslink_domain: docs.dnslinks.example.com
   ```

The action updates `_dnslink.docs.dnslinks.example.com`, and the CNAME redirects lookups from your main domain.

### Cloudflare TXT Record ID Auto-Discovery

When `cf_record_id` is not provided, the action automatically manages the DNS record:

1. Queries Cloudflare for existing TXT records named `_dnslink.{dnslink_domain}`
2. If no record exists: creates a new TXT record
3. If one record exists: updates it with the new CID
4. If multiple records exist: fails with an error asking you to manually remove duplicates

This eliminates the need to manually look up and configure the record ID.

## Usage

### Simple Workflow (No Fork PRs)

For repositories that don't accept PRs from forks, you can use a single workflow:

```yaml
name: Build and Deploy to IPFS

permissions:
  contents: read
  pull-requests: write
  statuses: write

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    outputs: # This exposes the CID output of the action to the rest of the workflow
      cid: ${{ steps.deploy.outputs.cid }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build project
        run: npm run build

      - name: Deploy to IPFS
        uses: ipshipyard/ipfs-deploy-action@v1
        id: deploy
        with:
          path-to-deploy: out
          storacha-key: ${{ secrets.STORACHA_KEY }}
          storacha-proof: ${{ secrets.STORACHA_PROOF }}
          github-token: ${{ github.token }}

      - name: Update DNSLink
        uses: ipshipyard/dnslink-action@v1
        if: github.ref == 'refs/heads/main' # only update the DNSLink on the main branch
        with:
          cid: ${{ steps.deploy.outputs.cid }}
          dnslink_domain: mydomain.com
          cf_zone_id: ${{ secrets.CF_ZONE_ID }}
          cf_auth_token: ${{ secrets.CF_AUTH_TOKEN }}
```

### Dual Workflows (For Secure Fork PRs)

For secure deployments of PRs from forks, use two separate workflows that pass artifacts between them:

**`.github/workflows/build.yml`** - Builds without secrets access:
```yaml
name: Build

permissions:
  contents: read

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  BUILD_PATH: 'out'  # Update this to your build output directory

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event_name == 'pull_request' && github.event.pull_request.head.sha || github.sha }}


      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build project
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: website-build-${{ github.run_id }}
          path: ${{ env.BUILD_PATH }}
          retention-days: 1
```

**`.github/workflows/deploy.yml`** - Deploys with secrets access:
```yaml
name: Deploy

permissions:
  contents: read
  pull-requests: write
  statuses: write

on:
  workflow_run:
    workflows: ["Build"]
    types: [completed]

env:
  BUILD_PATH: 'website-build'  # Directory where artifact from build.yml will be unpacked

jobs:
  deploy-ipfs:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    outputs: # This exposes the CID output of the action to the rest of the workflow
      cid: ${{ steps.deploy.outputs.cid }}
    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: website-build-${{ github.event.workflow_run.id }}
          path: ${{ env.BUILD_PATH }}
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ github.token }}

      - name: Deploy to IPFS
        uses: ipshipyard/ipfs-deploy-action@v1
        id: deploy
        with:
          path-to-deploy: ${{ env.BUILD_PATH }}
          storacha-key: ${{ secrets.STORACHA_KEY }}
          storacha-proof: ${{ secrets.STORACHA_PROOF }}
          github-token: ${{ github.token }}

      - name: Update DNSLink
        uses: ipshipyard/dnslink-action@v1
        if: github.event.workflow_run.head_branch == 'main' # only update for main branch
        with:
          cid: ${{ steps.deploy.outputs.cid }}
          dnslink_domain: mydomain.com
          cf_zone_id: ${{ secrets.CF_ZONE_ID }}
          cf_auth_token: ${{ secrets.CF_AUTH_TOKEN }}
          github_token: ${{ github.token }}
          set_github_status: true
```

## FAQ

- Why not use an infrastructure-as-code tool like [OctoDNS](https://github.com/octodns/octodns) or [DNSControl](https://github.com/StackExchange/dnscontrol)?
  - You can! Those are great tools.
- How can I safely build on PRs from forks?
  - Use the two-workflow pattern shown above. The build workflow runs on untrusted fork code without secrets access, while the deploy workflow only runs after a successful build and has access to secrets but never executes untrusted code. This pattern uses GitHub's `workflow_run` event to securely pass artifacts between workflows.
