# Static Sites — GitHub Pages Portfolio

Zero-cost static website hosting using GitHub Pages, with Cloudflare for DNS and email forwarding.

## Architecture

```
Domain Registrar (annual renewal — only cost)
        │
        ▼
   Cloudflare (free)
   ├── DNS ──────────────► GitHub Pages (free hosting)
   ├── Email Forwarding ──► Gmail (free)
   └── SSL ──────────────► Automatic via GitHub + Cloudflare
```

**Total hosting cost per site: $0/month**

## Managed Sites

<!-- Claude Code: Add new rows here when deploying a site. Remove rows when decommissioning. -->

| Domain | Repo | Status | Purpose | Last Updated |
|--------|------|--------|---------|--------------|
| *(example.com)* | *(example-com)* | *(live/pending)* | *(Brief description)* | *(YYYY-MM-DD)* |

## How to Use

### Add a new site (from scratch)

Open Claude Code in this directory and say:

> "Create a site for **[domain.com]**. [Describe what the site should be, who it's for, what content/sections to include, color preferences, etc.]"

### Add a new site (migrate existing)

> "Migrate the site at **[https://domain.com]** to GitHub Pages."

### Add a new site (migrate + redesign)

> "Migrate content from **[https://domain.com]** but redesign it as [describe new design]."

### Update an existing site

> "Update **[domain.com]** — [describe the changes]."

### Check status of all sites

> "Show me the status of all my sites."

## After Deployment: Cloudflare DNS Setup

After Claude Code deploys a new site, add these DNS records in the Cloudflare dashboard for that domain:

| Type  | Name | Content                          | Proxy Status |
|-------|------|----------------------------------|--------------|
| CNAME | @    | `<your-github-username>.github.io` | Proxied (orange) |
| CNAME | www  | `<your-github-username>.github.io` | Proxied (orange) |

**Also:** Remove any old A records or other records pointing to previous hosting.

## File Structure Per Site

```
domain-com/
├── index.html       # Main page (often the only page)
├── style.css        # Styles (or inlined in index.html)
├── CNAME            # GitHub Pages custom domain config
└── assets/          # Images, fonts, other static files
```

## Tools Required on the Development Machine

| Tool | Purpose | Install |
|------|---------|---------|
| `git` | Version control & deployment | Usually pre-installed |
| `gh` | GitHub CLI — repo & Pages management | `sudo apt install gh` or [cli.github.com](https://cli.github.com) |
| `wget` | Site mirroring for migrations | Usually pre-installed |
| `python3` | Local preview server | Usually pre-installed |

## Local Preview

```bash
cd ~/projects/sites/<domain-hyphenated>
python3 -m http.server 8080
# Visit http://localhost:8080 or use SSH tunnel
```

## Notes

- Each site folder is an independent Git repo (this parent directory is not)
- GitHub Pages auto-deploys on every `git push` to `main` (~60 seconds)
- Cloudflare proxy (orange cloud) is compatible with GitHub Pages
- SSL is fully automatic — no cert management needed
- Sites are static HTML/CSS/JS only — no server-side processing
- All repos are **public** on GitHub
