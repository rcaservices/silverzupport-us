# CLAUDE.md — Universal Static Site Manager

## What This Project Does

This system manages a portfolio of static websites deployed for free on GitHub Pages, with Cloudflare handling DNS and email forwarding. Each domain gets its own subdirectory and its own GitHub repo.

## How the User Interacts

The user will provide only two things:

1. **The domain name** (e.g., `example.com`)
2. **One or both of:**
   - A description of what the site should look like and contain
   - A URL of an existing site to migrate content from

Everything else — directory creation, site building, git setup, GitHub repo creation, Pages deployment, and DNS instructions — is handled by following the workflows below.

## Directory Convention

```
~/projects/sites/
├── CLAUDE.md              # This file
├── README.md              # Project overview and site registry
├── example-com/           # Domain with dots replaced by hyphens
│   ├── index.html
│   ├── style.css
│   ├── CNAME
│   └── assets/
├── another-domain-com/
│   └── ...
```

- Domain-to-folder naming: replace all dots with hyphens (e.g., `my.domain.com` → `my-domain-com`)
- Each site folder is its own independent Git repo
- The parent `~/projects/sites/` directory is NOT a Git repo
- GitHub repo name matches the folder name

## Environment Setup

On first run or if tools are missing, verify prerequisites:

```bash
# Check GitHub CLI is installed and authenticated
gh auth status || echo "SETUP NEEDED: Install gh CLI and run 'gh auth login'"

# Check git is configured
git config user.name || echo "SETUP NEEDED: Run 'git config --global user.name \"Your Name\"'"
git config user.email || echo "SETUP NEEDED: Run 'git config --global user.email \"you@email.com\"'"

# Get GitHub username (used throughout workflows)
GH_USER=$(gh api user --jq '.login')
echo "GitHub username: $GH_USER"
```

## Workflow 1: New Site from Description

**Trigger:** User provides a domain name and a description of what the site should be.

**Steps:**

1. **Derive the folder and repo name** from the domain:
   ```bash
   DOMAIN="example.com"
   REPO_NAME=$(echo "$DOMAIN" | tr '.' '-')
   SITE_DIR=~/projects/sites/$REPO_NAME
   ```

2. **Create the directory:**
   ```bash
   mkdir -p $SITE_DIR/assets
   ```

3. **Build the site** based on the user's description. Follow the Design Standards below.

4. **Initialize, commit, and push:**
   ```bash
   cd $SITE_DIR
   git init
   git add .
   git commit -m "Initial site for $DOMAIN"
   gh repo create $REPO_NAME --public --source=. --push
   ```

5. **Enable GitHub Pages:**
   ```bash
   GH_USER=$(gh api user --jq '.login')
   gh api repos/$GH_USER/$REPO_NAME/pages \
     -X POST -f source.branch=main -f source.path=/ 2>/dev/null \
     || echo "Pages may already be enabled"
   ```

6. **Add the CNAME file:**
   ```bash
   echo "$DOMAIN" > CNAME
   git add CNAME
   git commit -m "Add custom domain CNAME for $DOMAIN"
   git push
   ```

7. **Output DNS instructions** for the user:
   ```
   ✅ Site deployed! Update Cloudflare DNS for [DOMAIN]:

   | Type  | Name | Content                  | Proxy |
   |-------|------|--------------------------|-------|
   | CNAME | @    | [GH_USER].github.io      | On    |
   | CNAME | www  | [GH_USER].github.io      | On    |

   Remove any existing A records pointing to old hosting.
   SSL will be provisioned automatically (allow up to 10 minutes).
   ```

8. **Update the site registry** in README.md — add a row to the Managed Sites table.

## Workflow 2: Migrate an Existing Site

**Trigger:** User provides a domain name and a URL to migrate from.

**Steps:**

1. **Derive folder/repo name** (same as Workflow 1, step 1).

2. **Mirror the existing site:**
   ```bash
   mkdir -p /tmp/mirror-$REPO_NAME
   wget --mirror --convert-links --adjust-extension --page-requisites \
     --no-parent --timeout=30 --tries=3 \
     -P /tmp/mirror-$REPO_NAME https://$DOMAIN 2>&1
   ```

3. **If wget fails** (403, timeout, auth error):
   - Try with `http://` instead of `https://`
   - Try with `www.` prefix
   - If all attempts fail, inform the user and suggest alternatives:
     - Provide files directly (SCP, upload)
     - Give SSH access to the hosting server
     - Switch to Workflow 1 (build fresh using the existing site as visual reference only)

4. **Clean up the mirrored files:**
   ```bash
   SITE_DIR=~/projects/sites/$REPO_NAME
   mkdir -p $SITE_DIR/assets

   # Move files from wget's nested directory structure to site root
   # wget creates: /tmp/mirror-name/domain.com/path/files
   # We need: ~/projects/sites/domain-com/index.html

   # Find the actual content directory
   MIRROR_ROOT=$(find /tmp/mirror-$REPO_NAME -name "index.html" -type f | head -1 | xargs dirname)

   if [ -n "$MIRROR_ROOT" ]; then
     cp -r $MIRROR_ROOT/* $SITE_DIR/
   fi
   ```

5. **Strip server-side and CMS artifacts:**
   - Delete: `*.php`, `.htaccess`, `wp-admin/`, `wp-includes/`, `wp-content/plugins/`, `xmlrpc.php`, `wp-login.php`
   - Delete: any server config files (`.nginx.conf`, `web.config`)
   - Keep: HTML, CSS, JS, images, fonts, and other static assets
   - Move images and media into `assets/` if not already organized

6. **Fix references:**
   - Convert absolute URLs (`https://domain.com/path`) to relative paths (`./path`)
   - Remove broken references to deleted server-side files
   - Fix image paths if files were moved to `assets/`
   - Remove any inline analytics or tracking scripts unless user requests otherwise

7. **Replace dynamic elements with static alternatives:**
   - Contact forms → `mailto:` link using the domain's email address
   - Comment sections → Remove
   - Search functionality → Remove
   - Login forms → Remove

8. **Review the result:**
   - Verify `index.html` exists at root
   - Check that CSS/JS references are valid
   - Confirm images load (paths are correct)

9. **Deploy** — follow Workflow 1, steps 4-8.

10. **Clean up temp files:**
    ```bash
    rm -rf /tmp/mirror-$REPO_NAME
    ```

## Workflow 3: Update an Existing Site

**Trigger:** User asks to change something on an existing site.

**Steps:**

1. **Identify the site:**
   ```bash
   DOMAIN="example.com"
   REPO_NAME=$(echo "$DOMAIN" | tr '.' '-')
   SITE_DIR=~/projects/sites/$REPO_NAME
   cd $SITE_DIR
   ```

2. **Make the requested changes** to the HTML/CSS/asset files.

3. **Commit and push:**
   ```bash
   git add .
   git commit -m "Update: <brief description of changes>"
   git push
   ```

4. **Confirm to the user:** "Changes pushed. The site will be live at https://[DOMAIN] within ~60 seconds."

## Workflow 4: Check Status of a Site

```bash
DOMAIN="example.com"
REPO_NAME=$(echo "$DOMAIN" | tr '.' '-')
GH_USER=$(gh api user --jq '.login')

echo "=== Repo ==="
gh repo view $GH_USER/$REPO_NAME --json name,url,homepageUrl 2>/dev/null || echo "Repo not found"

echo "=== Pages Status ==="
gh api repos/$GH_USER/$REPO_NAME/pages --jq '.status' 2>/dev/null || echo "Pages not configured"

echo "=== Recent Commits ==="
cd ~/projects/sites/$REPO_NAME 2>/dev/null && git log --oneline -5 || echo "Local directory not found"

echo "=== Live Site Check ==="
curl -s -o /dev/null -w "HTTP %{http_code}" https://$DOMAIN 2>/dev/null || echo "Site not responding"
```

## Workflow 5: List All Managed Sites

```bash
echo "=== Managed Sites ==="
for dir in ~/projects/sites/*/; do
  if [ -f "$dir/CNAME" ]; then
    domain=$(cat "$dir/CNAME")
    repo=$(basename "$dir")
    echo "  $domain → $repo"
  fi
done
```

## Design Standards

When building or redesigning sites, follow these principles:

### Structure
- **Single HTML file** preferred for simple sites (inline CSS in a `<style>` tag is fine)
- Separate `style.css` file only if CSS exceeds ~100 lines
- All images and media in `assets/` subdirectory
- Always include a `CNAME` file matching the domain

### Visual Design
- **Mobile-first responsive layout** — must look good on phones, tablets, and desktop
- Clean, modern, professional appearance
- Good use of whitespace
- Readable typography (minimum 16px body text)
- Consistent color scheme (derive from user's description or branding if available)
- No heavy frameworks unless specifically requested — vanilla HTML/CSS is preferred

### Content
- Proper semantic HTML5 (`<header>`, `<main>`, `<section>`, `<footer>`)
- All images must have descriptive `alt` attributes
- Include a `<meta name="viewport">` tag for responsive behavior
- Include `<meta name="description">` for SEO
- Include a `<title>` tag matching the site's purpose
- Contact section should use the domain's email address (forwarded via Cloudflare)
- Include a copyright line in the footer with the current year

### Performance
- No external CDN dependencies unless truly necessary
- Optimize/compress images before adding to `assets/`
- Minimal or no JavaScript for simple sites
- Total page weight target: under 500KB

### What NOT to Include
- No analytics or tracking scripts (unless user requests)
- No cookie banners (static sites don't need them)
- No server-side code (PHP, Python, etc.)
- No build tools or package managers (no node_modules, no npm)
- No sensitive information (API keys, passwords, private data)
- No placeholder Lorem Ipsum text — use real content or ask the user

## Important Reminders

- **All GitHub repos are PUBLIC** — confirm with user before creating if not already established
- **GitHub username** is retrieved dynamically: `gh api user --jq '.login'`
- **DNS is managed by the user** in Cloudflare — always provide exact records needed
- **After initial DNS setup**, SSL provisioning takes ~10 minutes (up to 24 hours in rare cases)
- **Cloudflare proxy (orange cloud) ON** is compatible with GitHub Pages
- **Always update the README.md site registry** when adding or removing a site
