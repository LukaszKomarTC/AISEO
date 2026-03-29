# Hotel Partner Landing Page — Claude Code Spec

## Overview

Build a WordPress landing page for a partner hotel on a fresh Plesk VPS.
The site will showcase Tossa Ops cycling tour services and redirect hotel
clients to the main booking site for reservations.

All credentials are loaded from `.env` (never hardcoded).

---

## Environment

Create a `.env` file in the project root (gitignored) with the following:

```env
# Plesk
PLESK_API_URL=https://YOUR_VPS_IP:8443/api/v2
PLESK_API_KEY=your_plesk_api_key

# SSH (for WP-CLI)
SSH_HOST=YOUR_VPS_IP
SSH_USER=root
SSH_KEY_PATH=~/.ssh/id_rsa

# Site
HOTEL_DOMAIN=hotel-partner.com          # domain for the new WP site
HOTEL_NAME=Hotel Partner Name
HOTEL_SUBDOMAIN=                        # optional: e.g. partner.lukaszkomar.com

# WordPress
WP_ADMIN_USER=admin
WP_ADMIN_PASSWORD=generate_strong_password
WP_ADMIN_EMAIL=your@email.com
WP_SITE_TITLE=Hotel Partner - Cycling Tours

# Database (Claude Code will create these via Plesk API)
DB_NAME=hotel_partner_db
DB_USER=hotel_partner_user
DB_PASSWORD=generate_strong_password

# Tossa Ops booking URLs (update per service)
BOOKING_URL_MAIN=https://lukaszkomar.com/book
BOOKING_URL_GUIDED=https://lukaszkomar.com/guided-tours
BOOKING_URL_RENTALS=https://lukaszkomar.com/bike-rentals
BOOKING_URL_TRANSFERS=https://lukaszkomar.com/transfers
```

---

## Phase 1 — VPS & Domain Setup (Plesk API)

Use the Plesk REST API (`PLESK_API_URL`) with header:
`X-API-Key: PLESK_API_KEY`

### Steps

1. **Create subscription/domain**
   - `POST /api/v2/domains`
   - Set domain to `HOTEL_DOMAIN`
   - Assign to default service plan

2. **Create database**
   - `POST /api/v2/databases`
   - Name: `DB_NAME`, type: `mysql`
   - Linked to the domain created above

3. **Create database user**
   - `POST /api/v2/databases/{id}/users`
   - User: `DB_USER`, password: `DB_PASSWORD`
   - Grant all privileges on `DB_NAME`

4. **Install WordPress via WP Toolkit**
   - `POST /api/v2/modules/wp-toolkit/wordpress`
   - Set admin credentials from env
   - Set DB credentials from env
   - Set site title from `WP_SITE_TITLE`

5. **Issue SSL certificate**
   - `POST /api/v2/domains/{id}/ssl-certificates/letsencrypt`
   - Enable HTTPS redirect

> Verify each step returns 200/201 before proceeding. Log errors clearly.

---

## Phase 2 — WordPress Configuration (SSH + WP-CLI)

Connect via SSH using `SSH_HOST`, `SSH_USER`, `SSH_KEY_PATH`.
All WP-CLI commands run from the WordPress root directory on the server.

### Steps

1. **Verify WP install**
   ```bash
   wp core is-installed
   ```

2. **Install and activate theme**
   ```bash
   wp theme install generatepress --activate
   ```

3. **Install required plugins**
   ```bash
   wp plugin install advanced-custom-fields --activate
   wp plugin install classic-editor --activate
   wp plugin install redirection --activate
   ```

4. **Remove default content**
   ```bash
   wp post delete 1 2 --force        # sample page, hello world
   wp widget reset --all
   ```

5. **Create pages**
   ```bash
   wp post create --post_type=page --post_title='Home' --post_status=publish --post_name=home
   wp post create --post_type=page --post_title='Our Services' --post_status=publish --post_name=services
   wp post create --post_type=page --post_title='Contact' --post_status=publish --post_name=contact
   ```

6. **Set homepage**
   ```bash
   wp option update show_on_front page
   wp option update page_on_front $(wp post list --post_type=page --post_status=publish --field=ID --name=home)
   ```

7. **Set permalink structure**
   ```bash
   wp rewrite structure '/%postname%/' --hard
   ```

---

## Phase 3 — Landing Page Content (WP-CLI + PHP template)

### Homepage template

Create a custom page template at:
`wp-content/themes/generatepress-child/templates/landing.php`

The template must include the following sections:

#### 1. Hero
- Hotel name + tagline: *"Cycling adventures, starting from your door"*
- Full-width background image (placeholder: use Unsplash cycling/landscape)
- Primary CTA button → `BOOKING_URL_MAIN`

#### 2. Services Grid
Three cards, each with icon, title, short description, and CTA button:

| Service | Description | CTA URL |
|---|---|---|
| Guided Tours | Expert-led cycling routes around the area | `BOOKING_URL_GUIDED` |
| Bike Rentals | Quality bikes for independent exploration | `BOOKING_URL_RENTALS` |
| Transfers | Door-to-door transfers for you and your bike | `BOOKING_URL_TRANSFERS` |

All CTA buttons open in `_blank` with `rel="noopener noreferrer"`.

#### 3. Why Tossa Ops
Short trust-building section: 3 bullet points (local expertise, safety, flexibility).
Keep it concise — hotel clients are already warm leads.

#### 4. Footer CTA
Full-width banner: *"Ready to ride?"* + large button → `BOOKING_URL_MAIN`

#### 5. Footer
- Hotel name + logo placeholder
- Link back to hotel's main website (leave as `#` placeholder)
- Tossa Ops branding: *"Cycling services by Tossa Ops"* + link to `lukaszkomar.com`

### Design requirements
- Mobile-first, fully responsive
- Clean, minimal aesthetic — matches a hotel context
- No heavy page builders — pure HTML/CSS in the template
- Use Google Fonts (load via `wp_enqueue_style`)
- Enqueue custom CSS via child theme `functions.php`

### Apply the template to the homepage
```bash
wp post meta update $(wp post list --post_type=page --field=ID --name=home) _wp_page_template 'templates/landing.php'
```

---

## Phase 4 — Child Theme Setup

Create a GeneratePress child theme at:
`wp-content/themes/generatepress-child/`

Files to create:
- `style.css` — theme header + all custom styles
- `functions.php` — enqueue parent + child styles, Google Fonts
- `templates/landing.php` — landing page template (see Phase 3)

Activate the child theme:
```bash
wp theme activate generatepress-child
```

---

## Phase 5 — Final Checks

Run these verifications via SSH/WP-CLI:

```bash
# WP health
wp doctor check --all

# Confirm homepage is set
wp option get page_on_front

# Confirm permalink
wp option get permalink_structure

# Check active theme
wp theme list --status=active

# Check active plugins
wp plugin list --status=active
```

Also verify via HTTP:
- `https://HOTEL_DOMAIN` loads without errors
- SSL cert is valid
- All CTA buttons link to correct `BOOKING_URL_*` values
- Site is mobile responsive

---

## File Structure (local repo)

```
hotel-partner-landing/
├── .env                    # credentials (gitignored)
├── .env.example            # template with placeholders
├── .gitignore
├── SPEC.md                 # this file
├── scripts/
│   ├── 01-plesk-setup.sh       # Phase 1: Plesk API calls
│   ├── 02-wp-configure.sh      # Phase 2: WP-CLI setup
│   ├── 03-content.sh           # Phase 3: content + pages
│   └── 04-verify.sh            # Phase 5: health checks
└── theme/
    └── generatepress-child/
        ├── style.css
        ├── functions.php
        └── templates/
            └── landing.php
```

---

## Notes for Claude Code

- Always verify Plesk API responses before proceeding to the next phase
- If WP Toolkit API is unavailable, fall back to installing WP via WP-CLI over SSH:
  `wp core download && wp core config && wp core install`
- Do not expose `.env` values in logs or output
- If a step fails, log the error with the phase number and stop — do not continue to dependent steps
- All scripts should be idempotent where possible (safe to re-run)
