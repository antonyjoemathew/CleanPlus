# CleanPlus Laundry — Website

The live site for **CleanPlus Laundry** (Kakkanad, Kochi) — a prerendered, fully
static bundle. Deployed to an Azure VM by GitHub Actions on every push to `main`.

Live at <https://www.cleanpluslaundry.in/>.

## What's in this repo

This is **build output, not source.** The repo root is the web root:

```
index.html            Home
about/index.html      About
services/index.html   Services
contact/index.html    Contact
assets/               Hashed JS, CSS and images
favicon.png, icon-192.png, icon-512.png, apple-touch-icon.png
share-card.png        Open Graph / Twitter card (1200x630)
site.webmanifest      PWA manifest
robots.txt
```

No build step is needed — the deploy copies these files as-is.

All asset URLs are **root-relative** (`/assets/...`), so the site must be served
from the **domain root**, which is what `cleanpluslaundry.in` does. It is no
longer built for the `/cleanpluslaundry/` subpath that `pingee.app` used.

## Deployment

`.github/workflows/deploy.yml` runs on every push to `main`, and can also be
triggered manually from the **Actions** tab (**Run workflow**).

What it does:

1. Checks out the repo.
2. `rsync`s the site files to `/tmp/cleanpluslaundry-new/` on the VM
   (excluding `.git`, `.github`, `.gitignore`, `README.md`).
3. Over SSH: clears `/var/www/cleanpluslaundry`, copies the new files in, sets
   ownership to `www-data` and permissions to `755` (dirs) / `644` (files),
   removes the staging folder, then `nginx -t && systemctl reload nginx`.

### Required GitHub secrets

Under **Settings → Secrets and variables → Actions → Secrets → New repository
secret**. Same four names as the Pingee-Web repo — only the values differ, since
this is a different VM.

| Secret | What it is | Example |
| --- | --- | --- |
| `AZURE_VM_HOST` | Public IP or hostname of the Azure VM hosting this site | `20.244.xx.xx` |
| `AZURE_VM_PORT` | SSH port on that VM | `22` |
| `AZURE_VM_USER` | SSH user to connect as — must be able to `sudo` without a password prompt | `azureuser` |
| `AZURE_VM_SSH_KEY` | **Private** SSH key for that user, pasted whole — including the `-----BEGIN ... PRIVATE KEY-----` and `-----END ... PRIVATE KEY-----` lines. Must have no passphrase. | *(multi-line key)* |

The matching **public** key must be in `~/.ssh/authorized_keys` for
`AZURE_VM_USER` on the VM.

Because the deploy runs `sudo` non-interactively, that user needs passwordless
sudo — e.g. in `/etc/sudoers.d/deploy`:

```
azureuser ALL=(ALL) NOPASSWD: ALL
```

### Deploy paths (not secrets)

Plain `env:` values at the top of `.github/workflows/deploy.yml` — edit the file
if the server layout changes:

| Variable | Default | Purpose |
| --- | --- | --- |
| `DEPLOY_PATH` | `/var/www/cleanpluslaundry` | Folder nginx serves this site from. Its own dedicated folder, so the other sites on the VM are untouched. |
| `STAGING_PATH` | `/tmp/cleanpluslaundry-new` | Temporary upload folder, deleted after a successful deploy. |

## nginx

The site is its own vhost at the domain root, reading from its dedicated folder:

```nginx
server {
    listen 443 ssl http2;
    server_name www.cleanpluslaundry.in cleanpluslaundry.in;

    root /var/www/cleanpluslaundry;
    index index.html;

    # Every route is prerendered as its own folder, so a directory request
    # resolves to that folder's index.html. Return a real 404 for anything
    # missing — falling back to /index.html makes a missing stylesheet or image
    # answer 200 with HTML, which is what broke the styling before.
    location / {
        try_files $uri $uri/ $uri/index.html =404;
    }

    # Filenames are content-hashed, so cache them hard.
    location ^~ /assets/ {
        try_files $uri =404;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # ssl_certificate / ssl_certificate_key managed by certbot
}
```

Apply with `sudo nginx -t && sudo systemctl reload nginx`.

Worth adding if it isn't there yet: a 301 from `cleanpluslaundry.in` to
`www.cleanpluslaundry.in`. Both currently answer 200, which splits crawlers'
view of the same pages — the canonical tags say `www`.

## Updating the site

The React / TanStack Start **source** for this site lives next to this repo at
`../cleanplus-laundry-src` (git remote `antonyjoemathew/suds-and-say-hello`).

```bash
cd ../cleanplus-laundry-src
bun install          # first time only
bun run build        # prerenders into .output/public
```

Then copy the result over this repo's root and push:

```bash
cd ../CleanPlusLaundry
find . -mindepth 1 -maxdepth 1 ! -name .git ! -name .github ! -name .gitignore ! -name README.md -exec rm -rf {} +
cp -R ../cleanplus-laundry-src/.output/public/. ./
git add -A && git commit -m "Rebuild the site" && git push
```

Clearing first matters: asset filenames are content-hashed, so a plain copy
would leave the previous build's chunks behind.

Domain-related settings in the source, if the site ever moves again:

- `vite.config.ts` — `BASE_PATH` (`"/"` for a domain root)
- `src/lib/seo.ts` — `BASE_PATH` and `DEFAULT_SITE_URL` (the canonical domain
  used for `rel=canonical`, `og:url` and the share card URL; overridable at
  build time with `VITE_SITE_URL`)
- `public/site.webmanifest` — `start_url`, `scope`, icon paths

## Brand

- Primary green: `#69bc50` (`theme-color` and manifest `theme_color`)
- Logo: the "C + plus" mark — `assets/logo-mark-*.png`
- Type: Outfit (headings) and Figtree (body), loaded from Google Fonts
