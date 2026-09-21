# AUCTIQ Demo Worker

Separate, fully-isolated demo version of Auction Eye for trial users.

- **Database**: `auctiq-demo-db` (separate from production)
- **Subdomain**: `demo.lotstoknow.com`
- **Trial Duration**: 14 days (auto-cleanup after expiry)
- **Features**: Full Auction Eye except Lot Pairing

## Deployment

Automatic via GitHub Actions. Push to main to deploy.

Requires secrets:
- `CLOUDFLARE_API_TOKEN`
- `CLOUDFLARE_ACCOUNT_ID`
- `ANTHROPIC_API_KEY`
# Demo app deployment triggered
# Deployment with updated Cloudflare token
