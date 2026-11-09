# CleanPlus Laundry — Website

The live site for **CleanPlus Laundry** (Kakkanad, Kochi) — a prerendered, fully
static bundle. Deployed to an Azure VM by GitHub Actions on every push to `main`.

Currently live at <https://pingee.app/cleanpluslaundry/>.

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

> **Important:** every asset URL inside the HTML is absolute under
> `/cleanpluslaundry/` (and `site.webmanifest` sets `start_url` / `scope` to
> `/cleanpluslaundry/`). The site must therefore be served at the URL path
> **`/cleanpluslaundry/`**, not at the domain root. The nginx block below does
> exactly that.

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

Add these under **Settings → Secrets and variables → Actions → Secrets →
New repository secret**. Same four names as the Pingee-Web repo — only the
values differ (this is a different VM / IP).

| Secret | What it is | Example |
| --- | --- | --- |
| `AZURE_VM_HOST` | Public IP or hostname of the Azure VM hosting this site | `20.244.xx.xx` |
| `AZURE_VM_PORT` | SSH port on that VM | `22` |
| `AZURE_VM_USER` | SSH user to connect as — must be able to `sudo` without a password prompt | `azureuser` |
| `AZURE_VM_SSH_KEY` | **Private** SSH key for that user, pasted whole — including the `-----BEGIN ... PRIVATE KEY-----` and `-----END ... PRIVATE KEY-----` lines. Must have no passphrase. | *(multi-line key)* |

The matching **public** key must be in `~/.ssh/authorized_keys` for
`AZURE_VM_USER` on the VM.

Because the deploy runs `sudo` non-interactively, that user needs passwordless
sudo — e.g. `/etc/sudoers.d/deploy`:

```
azureuser ALL=(ALL) NOPASSWD: /bin/mkdir, /bin/rm, /bin/cp, /bin/chown, /usr/bin/find, /bin/ls, /usr/sbin/nginx, /bin/systemctl
```

(or simply `azureuser ALL=(ALL) NOPASSWD: ALL` if that's how the Pingee VM is
already set up).

### Deploy paths (not secrets)

These are plain `env:` values at the top of `.github/workflows/deploy.yml` —
edit the file if the server layout differs:

| Variable | Default | Purpose |
| --- | --- | --- |
| `DEPLOY_PATH` | `/var/www/cleanpluslaundry` | Final folder served by nginx. Its own dedicated folder, so other sites on the VM are untouched. |
| `STAGING_PATH` | `/tmp/cleanpluslaundry-new` | Temporary upload folder, deleted after a successful deploy. |

## One-time VM setup

```bash
sudo mkdir -p /var/www/cleanpluslaundry
sudo chown -R www-data:www-data /var/www/cleanpluslaundry
```

Then serve that folder at `/cleanpluslaundry/` in the site's nginx server block:

```nginx
# The URL prefix and the folder name match, so plain `root` resolves
# /cleanpluslaundry/<x> -> /var/www/cleanpluslaundry/<x> with no alias quirks.

location = /cleanpluslaundry {
    return 301 /cleanpluslaundry/;
}

location /cleanpluslaundry/ {
    root /var/www;
    index index.html;

    # Clean URLs: /cleanpluslaundry/about/ -> about/index.html
    try_files $uri $uri/ =404;
}

# Filenames are content-hashed, so cache them hard.
location ^~ /cleanpluslaundry/assets/ {
    root /var/www;
    try_files $uri =404;
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

Apply with `sudo nginx -t && sudo systemctl reload nginx`.

## Updating the site

The React/TanStack Start **source** for this site is a separate project
(`cleanplus-laundry`, pushed to `antonyjoemathew/suds-and-say-hello`). To change
content or design: edit there, run `bun run build`, then copy the contents of
`.output/public/` over this repo's root and push.

Two things to remember when the site moves to its own domain:

- `vite.config.ts` in the source sets `BASE_PATH = "/cleanpluslaundry/"`. Serving
  at a domain root means changing that and rebuilding.
- The Open Graph tags and the manifest icons still point at
  `https://pingee.app/cleanpluslaundry/...`. Those are absolute by design (social
  scrapers need them) and only affect link previews — update them in the source
  and rebuild once a final domain is decided.

## Brand

- Primary green: `#69bc50` (`theme-color` and manifest `theme_color`)
- Logo: the "C + plus" mark — `assets/logo-mark-*.png`
- Type: Outfit (headings) and Figtree (body), loaded from Google Fonts
