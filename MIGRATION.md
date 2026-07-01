# Migration: GitHub Pages → Azure Static Web Apps (Free)

Prepared changes in this repo:

- `.github/workflows/azure-swa-deploy.yml` — builds Jekyll (Ruby 3.3, same as your current workflow) and deploys the prebuilt `_site` to Azure SWA. Includes PR staging environments.
- `staticwebapp.config.json` — 404 page, security headers. Jekyll copies it into `_site` automatically.
- `_config.yml` — `url` changed to `https://marcoheijkoop.nl` (was pointing at github.io).

## Step 1 — Create the Static Web App

1. [Azure Portal](https://portal.azure.com) → Create a resource → **Static Web App**.
2. Basics:
   - Resource group: new, e.g. `rg-website`
   - Name: e.g. `marcoheijkoop-nl`
   - Plan: **Free**
   - Region: **West Europe**
   - Source: **Other** (not GitHub — this avoids Azure committing its own workflow file; we use ours)
3. Create, then open the resource → **Overview** → click **Manage deployment token** → copy the token.

## Step 2 — Add the token to GitHub

GitHub repo → Settings → Secrets and variables → Actions → **New repository secret**:

- Name: `AZURE_STATIC_WEB_APPS_API_TOKEN`
- Value: the deployment token

## Step 3 — Push and verify

Push this repo (the new workflow triggers on push to `main`). Check Actions tab → "Deploy to Azure Static Web Apps" succeeds. Then open the `*.azurestaticapps.net` URL from the Azure Overview page and verify the site works. GitHub Pages keeps serving marcoheijkoop.nl meanwhile — zero downtime.

## Step 4 — Custom domain (keep Cloudflare DNS)

In Azure: Static Web App → **Custom domains** → **+ Add** → Custom domain on other DNS.

For the apex `marcoheijkoop.nl`:

1. Enter `marcoheijkoop.nl`, choose **TXT** validation, click **Generate code**, copy it.
2. In Cloudflare DNS, add a **TXT** record with the exact host name Azure shows (typically `_dnsauth`) and the generated value. **DNS only** (grey cloud).
3. Wait for Azure to show "Validated" (minutes to an hour).
4. In Cloudflare, replace the current apex records (the GitHub Pages A records `185.199.108.153` etc., or CNAME) with:
   - Type: **CNAME**, Name: `@`, Target: your `<app>.azurestaticapps.net` hostname (no `https://`). Cloudflare flattens CNAMEs at the apex automatically.
   - Set it **DNS only** first, until Azure shows the domain as ready and HTTPS works.
5. If you also use `www.marcoheijkoop.nl`: add it in Azure as a second custom domain (free tier allows 2) and a CNAME `www` → `<app>.azurestaticapps.net`.

Azure issues a free SSL certificate automatically once the domain validates.

## Step 5 — Re-enable Cloudflare features (optional)

After HTTPS works on marcoheijkoop.nl:

- You can re-enable the orange cloud (proxy) on the CNAME records if you want Cloudflare caching/WAF.
- Cloudflare SSL/TLS mode: **Full (strict)**.
- Note: Azure SWA already serves via a global CDN with free SSL, so plain "DNS only" is also fine and one less layer to debug.

## Step 6 — Tear down GitHub Pages

Once the domain serves from Azure:

1. GitHub repo → Settings → Pages → disable ("Unpublish site") or set Source to None.
2. Delete the old workflows: `.github/workflows/pages-deploy.yml` and `.github/workflows/jekyll.yml`.
3. Delete the `CNAME` and `.nojekyll` files (GitHub Pages specific, unused by Azure).
4. This `MIGRATION.md` can be deleted too.

## Notes

- The workflow builds `_site` itself and deploys with `skip_app_build: true` — Azure's Oryx builder is bypassed, so the build is identical to your current GitHub Pages build (Ruby 3.3 + htmlproofer test).
- Free tier limits: 250 MB app size, 100 GB bandwidth/month, 2 custom domains — plenty for this site.
- PR staging: pull requests get a temporary preview URL, closed automatically on merge.
