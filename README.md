# alphax-mta-sts

Hosts the MTA-STS policy file for **alphax.ai**, deployed via Cloudflare Pages (auto-deploys from `main`).

## Why this repo exists

MTA-STS requires a policy file to be served at `https://mta-sts.alphax.ai/.well-known/mta-sts.txt` over valid HTTPS. Microsoft 365 (Exchange Online, which alphax.ai uses for mail) doesn't host this for you on a custom domain, so this repo + Cloudflare Pages fills that gap. TLS-RPT (the companion reporting record) needs no hosting, just a DNS TXT record.

## What's in this repo

- `.well-known/mta-sts.txt` - the MTA-STS policy served at the site root.
- `CNAME` - GitHub's file for its own Pages product; not used here since we deploy via Cloudflare Pages instead (kept for reference, harmless either way).

Current policy:

```
version: STSv1
mode: testing
mx: *.mail.protection.outlook.com
max_age: 604800
```

`mode: testing` means receivers report failures without enforcing TLS yet. Once TLS-RPT reports (see below) show no unexpected failures for a couple of weeks, switch to `mode: enforce` and bump the `id` value in the `_mta-sts.alphax.ai` DNS TXT record (see below) so resolvers pick up the change.

Note: the mx value above uses a wildcard (`*.mail.protection.outlook.com`) rather than the bare hostname, because Microsoft 365's real MX record for a tenant is tenant-specific (e.g. `alphax-ai.mail.protection.outlook.com`), not the literal `mail.protection.outlook.com`. Without the wildcard, MTA-STS validators (e.g. MXToolbox) fail with "No MTA-STS Policy MX Pattern Match" even though the record and policy file are otherwise valid. Whenever this file changes, bump the `id` value in the `_mta-sts.alphax.ai` TXT record too, so resolvers know to refetch the policy.

## Hosting: Cloudflare Pages

- Cloudflare account: `Craig.hamilton@naturealpha.ai`
- Pages project: `alphax-mta-sts` (deploys to `alphax-mta-sts.pages.dev`)
- Connected to this GitHub repo (Nature-Alpha-Ltd/alphax-mta-sts), scoped to this repo only - pushes to `main` auto-deploy.
- Custom domain `mta-sts.alphax.ai` added in the Pages project's Custom Domains tab, set up via "My DNS provider" (CNAME method) since alphax.ai's DNS is NOT on Cloudflare nameservers (it's on `atom.com`). Cloudflare issues and manages the TLS certificate for this subdomain automatically once the CNAME below is verified.

## DNS records required on alphax.ai

These live at alphax.ai's actual DNS provider (`ns1/ns2.atom.com`), not Cloudflare:

| Type | Name | Value |
|------|------|-------|
| CNAME | `mta-sts.alphax.ai` | `alphax-mta-sts.pages.dev` |
| TXT | `_mta-sts.alphax.ai` | `v=STSv1; id=1786110190643` |
| TXT | `_smtp._tls.alphax.ai` | `v=TLSRPTv1; rua=mailto:6511556403086@tls.dmarcly.com` |

The CNAME verifies the Cloudflare custom domain and serves the policy file. The two TXT records are what actually turn MTA-STS and TLS-RPT on for mail sent to alphax.ai - without them, the policy file being reachable does nothing.

## Verifying it's working

- Policy file: `curl https://mta-sts.alphax.ai/.well-known/mta-sts.txt` should return the policy above with a `200` (no redirects allowed by the MTA-STS spec).
- dmarcly's MTA-STS & TLS-RPT dashboard will start showing TLS-RPT report data a few days after the TXT records propagate.
- TXT record checks: `_mta-sts.alphax.ai` and `_smtp._tls.alphax.ai` via any DNS lookup tool once propagated (can take up to 24 hours).
