# DNSLink Action

This GitHub Action sets the [DNSLink](https://www.dnslink.dev/) TXT record for a given domain.

Supports the following DNS providers:

- Cloudflare
- DNSimple
- Gandi

This action is built and maintained by [Interplanetary Shipyard](http://ipshipyard.com/).
<a href="http://ipshipyard.com/"><img align="right" src="https://github.com/user-attachments/assets/39ed3504-bb71-47f6-9bf8-cb9a1698f272" /></a>

It's a [composite action](https://docs.github.com/en/actions/sharing-automations/creating-actions/about-custom-actions#composite-actions) that can be called as a step in your workflow to set the DNSLink TXT record for a given domain.

## Inputs

### Required Inputs

| Input                 | Description                                                                                                              |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `cid`                 | CID of the build to update the DNSLink for                                                                               |
| `dnslink_domain`      | Domain to update the DNSLink for e.g. if you set docs.ipfs.tech, the \_dnslink.docs.ipfs.tech TXT record will be updated |
| `cf_record_id`        | Cloudflare Record ID                                                                                                     |
| `cf_zone_id`          | Cloudflare Zone ID                                                                                                       |
| `cf_auth_token`       | Cloudflare API token                                                                                                     |
| `dnsimple_token`      | DNSimple API token                                                                                                       |
| `dnsimple_account_id` | DNSimple account ID                                                                                                      |
| `gandi_pat`           | Gandi Personal Authorization Token (Bearer auth)                                                                         |
| `gandi_rrset_name`    | Name of the record in the dnslink_domain (i.e. "mysubdomain", _dnslink. will be prepended)                               |

### Optional Inputs

| Input               | Description                                                  |
| ------------------- | ------------------------------------------------------------ |
| `set_github_status` | Set the GitHub commit status with the DNSLink domain and CID |
| `github_token`      | GitHub token                                                 |

## Usage

See the [IPNS Inspector](https://github.com/ipfs/ipns-inspector/blob/main/.github/workflows/build.yml) for a real-world example of this action in use.


Here's a basic example of how to use this action in your workflow:

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

      - uses: ipfs/ipfs-deploy-action@v0.3
        name: Deploy to IPFS
        id: deploy
        with:
          path-to-deploy: out
          storacha-key: ${{ secrets.STORACHA_KEY }}
          storacha-proof: ${{ secrets.STORACHA_PROOF }}
          github-token: ${{ github.token }}

      - uses: ipfs/dnslink-action@v0.1
        if: github.ref == 'refs/heads/main' # only update the DNSLink on the main branch
        name: Update DNSLink
        with:
          cid: ${{ steps.deploy.outputs.cid }} # The CID of the build to update the DNSLink for
          dnslink_domain: mydomain.com
          cf_record_id: ${{ secrets.CF_RECORD_ID }}
          cf_zone_id: ${{ secrets.CF_ZONE_ID }}
          cf_auth_token: ${{ secrets.CF_AUTH_TOKEN }}
```

## FAQ

- Why not use an infrastructure-as-code tool like [OctoDNS](https://github.com/octodns/octodns) or [DNSControl](https://github.com/StackExchange/dnscontrol)?
  - You can! Those are great tools.
