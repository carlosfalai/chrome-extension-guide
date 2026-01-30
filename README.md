# Chrome Extension Development Guide

**Reference guide for building Chrome extensions from scratch to Chrome Web Store publication.**

When working on Chrome extensions, reference this guide. For project-specific context, also check any project CLAUDE.md files.

---

## ⚠️ CRITICAL WARNINGS

### Image Generation Quality Check
**ALWAYS visually verify generated images before giving to user!**
- Open each generated image and check text is NOT cut off
- Verify all text is readable and centered
- Use `ctx.textAlign = 'center'` and calculate positions from center
- Use relative font sizes based on canvas dimensions (e.g., `height * 0.12`)
- NEVER use fixed pixel positions that may cause clipping

### Store Listing Text
**ALWAYS provide complete form field values including:**
- All dropdown selections (Category, Language, etc.)
- All checkboxes to check/uncheck
- All text fields with exact copy-paste content
- Permission justifications for each permission used

### Monetization — NO Fake Affiliate Links
**NEVER add placeholder affiliate links that don't actually pay.**
- No `?ref=authcode` or similar fake tracking codes
- No ads unless they actually generate revenue
- For donations: Use Stripe payment links (no platform fees beyond ~3%)
- For tips: Create a "Buy Me a Coffee" product in Stripe ($4.99) with optional message field
- Google AdSense does NOT work in Chrome extensions (CSP blocks it)

**Approved monetization methods:**
1. Stripe one-time payments (Pro tiers, Lifetime, donations)
2. Stripe subscriptions
3. Chrome Web Store payments (discontinued by Google in 2020)
4. Real affiliate programs (must have actual account with tracking ID)

### Support Pages Required
**EVERY Chrome extension project MUST have a dedicated support page.**
- Do NOT use custom email domains per project (no support@projectname.com)
- Use the standard support email: `instanthpi@gmail.com`
- Deploy support.html to Cloudflare Pages alongside landing page and privacy policy
- Support URL format: `https://[project].pages.dev/support.html`

**Files required for every project deployment:**
```
store-assets/
├── index.html              (landing page, copied from landing-page.html)
├── landing-page.html       (original landing page)
├── support.html            (dedicated support page)
├── privacy-policy.html     (privacy policy)
├── sitemap.xml             (for Google Search Console indexing)
├── google[xxx].html        (Google Search Console verification, if needed)
└── _redirects              (Cloudflare redirects, if needed)
```

**Sitemap template (sitemap.xml):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://[project].pages.dev/</loc>
    <lastmod>[YYYY-MM-DD]</lastmod>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>https://[project].pages.dev/support.html</loc>
    <lastmod>[YYYY-MM-DD]</lastmod>
    <priority>0.8</priority>
  </url>
  <url>
    <loc>https://[project].pages.dev/privacy-policy.html</loc>
    <lastmod>[YYYY-MM-DD]</lastmod>
    <priority>0.5</priority>
  </url>
</urlset>
```

---

## ⚠️ MANDATORY: Test Before Reporting Done

**NEVER tell the user a feature is complete without testing first!**

After implementing any change:

1. **Syntax Check** - Run Node.js syntax validation on all modified JS files:
   ```bash
   node --check background.js
   node --check content.js
   node --check popup.js
   ```

2. **JSON Validation** - Validate manifest.json:
   ```bash
   node -e "JSON.parse(require('fs').readFileSync('manifest.json'))"
   ```

3. **File Verification** - Confirm all required files exist and were modified:
   ```bash
   ls -la  # Check file sizes changed
   ```

4. **Playwright Testing** (when available) - Actually load the extension and verify behavior

**Testing Sequence:**
```
Implement Feature → Syntax Check → JSON Check → File Check → Tell User Done
```

**If syntax errors are found:** Fix them immediately before reporting to user.

---

## Extension Projects Location

`C:\Users\Carlos Faviel Font\`

## Development Workflow

1. **ALWAYS test with syntax checks after making changes** — `node --check *.js`
2. Always test with Playwright after making changes — launch Chrome with the extension loaded and verify behavior
3. Check console logs for errors (`page.on('console', ...)`)
4. Take screenshots to verify visual changes (blur, overlays, layout)
5. Test on multiple page types since DOM structure differs

---

## Browser Testing with Playwright

Playwright is installed globally with Chromium. Use it to test Chrome extensions, inspect pages, take screenshots, and debug UI issues from the terminal.

**Launch Chrome with an extension loaded:**
```js
const { chromium } = require('playwright');
const browser = await chromium.launchPersistentContext('/tmp/test-profile', {
  headless: false,
  args: [
    '--disable-extensions-except=C:\\Users\\Carlos Faviel Font\\[EXTENSION_FOLDER]',
    '--load-extension=C:\\Users\\Carlos Faviel Font\\[EXTENSION_FOLDER]'
  ]
});
const page = await browser.newPage();
await page.goto('https://example.com');
```

**Common tasks:**
- `await page.screenshot({ path: 'screenshot.png', fullPage: true })` — capture page state
- `await page.evaluate(() => document.querySelectorAll('.selector').length)` — check DOM
- `page.on('console', msg => console.log(msg.text()))` — capture console logs
- `await page.waitForSelector('.element')` — wait for elements

---

## Chrome Extension OAuth Setup (Firebase + Google Sign-In)

For extensions using Firebase Auth with Google Sign-In, follow this workflow:

**1. Create Firebase/GCP Project:**
```bash
gcloud projects create PROJECT_ID --name="Project Name"
gcloud services enable firestore.googleapis.com identitytoolkit.googleapis.com firebase.googleapis.com apikeys.googleapis.com --project=PROJECT_ID
gcloud firestore databases create --project=PROJECT_ID --location=nam5 --type=firestore-native
gcloud alpha services api-keys create --project=PROJECT_ID --display-name="Web Key"
```

**2. Generate Extension Key and ID:**
```js
const crypto = require('crypto');
const { publicKey } = crypto.generateKeyPairSync('rsa', {
  modulusLength: 2048,
  publicKeyEncoding: { type: 'spki', format: 'der' },
  privateKeyEncoding: { type: 'pkcs8', format: 'der' }
});
const manifestKey = publicKey.toString('base64');
const hash = crypto.createHash('sha256').update(publicKey).digest();
let extId = '';
for (let i = 0; i < 16; i++) {
  extId += String.fromCharCode(97 + (hash[i] >> 4));
  extId += String.fromCharCode(97 + (hash[i] & 0xf));
}
console.log('MANIFEST KEY:', manifestKey);
console.log('EXTENSION ID:', extId);
```

**3. Create OAuth Client (requires browser automation):**

Google blocks automated login, so use a persistent Playwright profile:

```js
const { chromium } = require('playwright');
const browser = await chromium.launchPersistentContext(
  'C:/Users/Carlos Faviel Font/chrome-gcp-profile',
  { channel: 'chrome', headless: false }
);
const page = browser.pages()[0] || await browser.newPage();
await page.goto('https://console.cloud.google.com/apis/credentials?project=PROJECT_ID');
```

After user signs in, automate OAuth client creation:
```js
await page.click('text=Create credentials');
await page.click('text=OAuth client ID');
await page.click('[data-value="CHROME_EXTENSION"]');
await page.fill('input[aria-label="Item 1"]', 'EXTENSION_ID_HERE');
await page.click('text=Create');
```

**4. Deploy Firestore Security Rules:**
```bash
ACCESS_TOKEN=$(gcloud auth print-access-token)
curl -X POST "https://firebaserules.googleapis.com/v1/projects/PROJECT_ID/rulesets" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "x-goog-user-project: PROJECT_ID" \
  -H "Content-Type: application/json" \
  -d '{"source":{"files":[{"name":"firestore.rules","content":"rules_version = '\''2'\'';\nservice cloud.firestore {\n  match /databases/{database}/documents {\n    match /users/{userId}/{document=**} {\n      allow read, write: if request.auth != null && request.auth.uid == userId;\n    }\n  }\n}"}]}}'
```

**5. Persistent GCP Browser Session:**
Maintain at: `C:/Users/Carlos Faviel Font/chrome-gcp-profile`

---

## Auth-Gated Data Protection (CRITICAL)

**All user data MUST be tied to authentication state.**

When a user signs out, their data should NOT be accessible locally - just like Chrome bookmarks aren't visible when signed out of Chrome. This protects privacy on shared devices.

### Implementation Pattern

**1. popup.html - Add auth-required overlay:**
```html
<!-- Auth Required Screen - Shown when NOT signed in -->
<div id="authRequired" style="display:none;">
  <div class="auth-icon">🔐</div>
  <h2>Sign in to Continue</h2>
  <p>Your data is private and secure. Sign in to access your profiles, progress, and settings.</p>
  <button class="auth-btn" id="authSignInBtn">
    <img src="https://www.google.com/favicon.ico" width="18" height="18" alt="">
    Sign in with Google
  </button>
  <p class="auth-privacy">
    <strong>Privacy First:</strong> All your data is tied to your account.
    When you sign out, nothing is accessible locally.
  </p>
</div>

<!-- Main App Content - Only visible when signed in -->
<div id="appContent">
  <!-- All your app content here -->
</div>
```

**2. popup.js - Auth check on load:**
```javascript
document.addEventListener('DOMContentLoaded', async () => {
  const settingsData = await chrome.runtime.sendMessage({ action: 'getSettings' });
  const settings = settingsData?.settings || {};
  const isSignedIn = !!settings?.user;

  const appContent = document.getElementById('appContent');
  const authRequired = document.getElementById('authRequired');

  if (!isSignedIn) {
    // NOT SIGNED IN: Hide all app content
    if (appContent) appContent.style.display = 'none';
    if (authRequired) authRequired.style.display = 'flex';
    setupSignInButton();
    return; // Don't load any user data
  }

  // SIGNED IN: Show app, load data
  if (appContent) appContent.style.display = 'block';
  if (authRequired) authRequired.style.display = 'none';

  // Now safe to load user data...
});
```

**3. background.js - Clear ALL data on sign-out:**
```javascript
async function googleSignOut() {
  try {
    const token = await chrome.identity.getAuthToken({ interactive: false });
    if (token?.token) {
      await chrome.identity.removeCachedAuthToken({ token: token.token });
    }

    // CRITICAL: Clear ALL user data on sign-out
    await chrome.storage.local.clear();

    // Restore minimal defaults (no user data)
    await chrome.storage.local.set({
      settings: { enabled: false, user: null }
    });

    return { success: true };
  } catch (err) {
    return { success: false, error: err.message };
  }
}
```

**4. Test coverage:**
```javascript
// Test auth-required screen shows when not signed in
const authVisible = await page.locator('#authRequired').isVisible();
const appHidden = await page.locator('#appContent').evaluate(
  el => getComputedStyle(el).display === 'none'
);
expect(authVisible && appHidden).toBe(true);
```

### Why This Matters

- **Shared devices:** Family computers, school computers, library computers
- **Privacy expectations:** Users expect sign-out to mean "hide my stuff"
- **Chrome behavior:** Matches how Chrome handles bookmarks, history when signed out
- **Trust:** Users trust that their data isn't visible to the next person

**NEVER store sensitive data locally without auth-gating.**

---

## Chrome Web Store Submission Checklist

When an extension is ready for Chrome Web Store submission, **proactively generate all required assets and provide all form fields** before the user uploads the zip.

### 1. Pre-Submission Assets to Generate

**Install dependencies:**
```bash
cd [EXTENSION_FOLDER]
npm init -y
npm install canvas playwright
```

**generate-icons.js:**
```js
const { createCanvas } = require('canvas');
const fs = require('fs');

function drawIcon(size) {
  const canvas = createCanvas(size, size);
  const ctx = canvas.getContext('2d');
  ctx.fillStyle = '#1a1a2e';
  ctx.fillRect(0, 0, size, size);
  // Draw your icon design here
  return canvas.toBuffer('image/png');
}

fs.writeFileSync('icon48.png', drawIcon(48));
fs.writeFileSync('icon128.png', drawIcon(128));
console.log('Generated icons');
```

**generate-promo.js (CORRECTED - centers text properly):**
```js
const { createCanvas } = require('canvas');
const fs = require('fs');

function drawPromo(width, height, filename) {
  const canvas = createCanvas(width, height);
  const ctx = canvas.getContext('2d');

  // Background gradient
  const grad = ctx.createLinearGradient(0, 0, width, height);
  grad.addColorStop(0, '#4f46e5');
  grad.addColorStop(1, '#7c3aed');
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, width, height);

  // ⚠️ IMPORTANT: Center all text to avoid clipping
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  const centerX = width / 2;

  // Use relative sizes based on canvas dimensions
  ctx.font = `${height * 0.18}px Arial`;
  ctx.fillStyle = '#ffffff';
  ctx.fillText('🔐', centerX, height * 0.22);

  ctx.font = `bold ${height * 0.12}px Arial, sans-serif`;
  ctx.fillText('[EXTENSION NAME]', centerX, height * 0.45);

  ctx.font = `${height * 0.06}px Arial, sans-serif`;
  ctx.fillStyle = '#e0e7ff';
  ctx.fillText('[Tagline]', centerX, height * 0.58);

  ctx.font = `${height * 0.045}px Arial, sans-serif`;
  ctx.fillStyle = '#c7d2fe';
  ctx.fillText('[Feature 1] • [Feature 2] • [Feature 3]', centerX, height * 0.70);

  ctx.font = `bold ${height * 0.04}px Arial, sans-serif`;
  ctx.fillStyle = '#ffffff';
  ctx.fillText('[Pricing info]', centerX, height * 0.85);

  fs.writeFileSync(filename, canvas.toBuffer('image/png'));
  console.log(`Generated: ${filename}`);
  console.log('⚠️  VERIFY: Open image and check text is not cut off!');
}

drawPromo(440, 280, 'promo-small-440x280.png');
drawPromo(1400, 560, 'promo-marquee-1400x560.png');
```

### 2. Manifest Requirements
- `description` must be ≤132 characters
- `name` should be clear and unique
- Include `icons` with 48px and 128px versions

---

## 3. COMPLETE Store Listing Form Fields

**Provide ALL of these to the user in chat, ready to copy-paste:**

### Store Listing Tab

```
=== DROPDOWN SELECTIONS ===
Language: English
Category: [Select ONE]
  - Productivity (most extensions)
  - Developer Tools (dev-focused)
  - Social & Communication
  - Shopping
  - Entertainment
  - Accessibility
  - Search Tools

=== SHORT DESCRIPTION (≤132 chars) ===
[One line description of what it does]

=== DETAILED DESCRIPTION (copy entire block) ===
[Extension Name] is [what it does in one sentence].

HOW IT WORKS
• [Step 1]
• [Step 2]
• [Step 3]

KEY FEATURES

🔐 [Feature Category 1]
- [Benefit 1]
- [Benefit 2]

⚡ [Feature Category 2]
- [Benefit 1]
- [Benefit 2]

PRICING
- FREE: [What's free]
- [Paid tier]: [What's included]

PRIVACY
[Brief privacy statement]

=== ASSETS TO UPLOAD ===
- Screenshot 1: store-assets/screenshot-1.png (1280x800)
- Small promo: store-assets/promo-small-440x280.png
- Marquee promo: store-assets/promo-marquee-1400x560.png
```

---

## 4. COMPLETE Privacy Practices Form Fields

```
=== SINGLE PURPOSE (≤1000 chars) ===
[Extension Name] is a [type] extension that [primary function].
It [secondary function if any].

=== PERMISSION JUSTIFICATIONS ===
(Copy the ones that apply to your extension)

storage: Stores user preferences and settings locally using chrome.storage.local and chrome.storage.sync.

clipboardWrite: Copies [data type] to clipboard when user clicks [action].

clipboardRead: Reads clipboard content to [purpose].

identity: Enables optional Google Sign-In for [purpose - e.g., cloud sync, license validation].

activeTab: Accesses current tab content only when user clicks the extension icon.

tabs: Required to [purpose - e.g., detect active tab URL, open new tabs].

host_permissions (if used): Required to [action] on [specific domains]. All processing happens locally.

=== REMOTE CODE ===
Select: "No, I am not using remote code"
(Unless you load .js from external URLs)

=== DATA USAGE CHECKBOXES ===
CHECK these if applicable:
☑️ User's email address (if Google sign-in used)
☑️ Authentication tokens (if Firebase/OAuth used)

DO NOT CHECK (unless actually collected):
☐ Personally identifiable information
☐ Health information
☐ Financial and payment information
☐ Authentication information (passwords)
☐ Personal communications
☐ Location
☐ Web history
☐ User activity (clicks, scrolls, etc.)
☐ Website content

=== CERTIFICATIONS ===
CHECK ALL THREE:
☑️ I certify this item does not contain any of the following: malware, malicious scripts, cryptocurrency miners, etc.
☑️ I certify this item complies with Chrome Web Store Developer Program Policies
☑️ I certify this item does not contain obfuscated code

=== PRIVACY POLICY URL ===
https://[project-name].pages.dev/privacy-policy.html
```

---

## 5. Landing Page Requirements

**ALWAYS include these sections on landing pages:**

1. **Hero** - Name, tagline, CTA button
2. **Features** - 4-6 feature cards with icons
3. **Pricing** - All tiers (Free, Paid, Premium)
4. **Broadcast/Ads Section** (if applicable) - Let users pay for visibility
5. **Footer** - Privacy policy link, contact

**If the extension has supporter/sponsor features, include:**
```html
<!-- Broadcast Section -->
<div class="broadcast">
  <h2>📢 Broadcast Your Message</h2>
  <p>Get your message seen by all users! Your custom message scrolls in the supporter credits.</p>
  <div class="price">$XX</div>
  <ul>
    <li>✓ Your message shown to all users</li>
    <li>✓ Includes clickable link</li>
    <li>✓ Runs forever</li>
  </ul>
  <a href="#">Get Broadcast Slot</a>
</div>
```

**Deploy to Cloudflare Pages:**
```bash
# Create index.html from landing page so root URL works
cp landing-page.html index.html
npx wrangler pages project create [project-name] --production-branch main
npx wrangler pages deploy . --project-name=[project-name]
```

**Resulting URLs:**
- Homepage: `https://[project].pages.dev`
- Support: `https://[project].pages.dev/support.html`
- Privacy: `https://[project].pages.dev/privacy-policy.html`

---

## 6. Privacy Policy Template

```html
<!DOCTYPE html>
<html><head><title>Privacy Policy - [NAME]</title></head>
<body>
<h1>Privacy Policy</h1>
<p>Last updated: [DATE]</p>
<h2>Overview</h2>
<p>[NAME] processes data locally in your browser.</p>
<h2>Data We Collect</h2>
<p>Locally stored: [list]. Never transmitted.</p>
<h2>Data We Do NOT Collect</h2>
<p>We do not track pages you visit or sell data.</p>
<h2>Contact</h2>
<p>Email: instanthpi@gmail.com</p>
</body></html>
```

---

## 7. Support Page Template

**Default support email for ALL projects:** `instanthpi@gmail.com`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Support - [NAME]</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
      background: linear-gradient(135deg, #4f46e5, #7c3aed);
      color: #fff;
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .container { max-width: 600px; margin: 0 auto; padding: 40px 20px; text-align: center; }
    h1 { font-size: 2.5em; margin-bottom: 16px; }
    p { color: #e0e7ff; line-height: 1.6; margin-bottom: 24px; }
    .email-box {
      background: rgba(255,255,255,0.1);
      padding: 32px;
      border-radius: 16px;
      backdrop-filter: blur(10px);
    }
    .email-link {
      display: inline-block;
      background: #fff;
      color: #4f46e5;
      padding: 16px 32px;
      font-size: 1.2em;
      font-weight: 600;
      border-radius: 12px;
      text-decoration: none;
    }
    .back-link { display: inline-block; margin-top: 24px; color: #e0e7ff; text-decoration: none; }
  </style>
</head>
<body>
  <div class="container">
    <h1>Support</h1>
    <p>Need help? Have a question, suggestion, or found a bug? We're here to help.</p>
    <div class="email-box">
      <p style="margin-bottom: 16px;">Contact us via email:</p>
      <a href="mailto:instanthpi@gmail.com" class="email-link">instanthpi@gmail.com</a>
    </div>
    <a href="/" class="back-link">&larr; Back to home</a>
  </div>
</body>
</html>
```

---

## 8. Landing Page Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>[NAME] - [Tagline]</title>
  <style>
    body { font-family: -apple-system, sans-serif; background: linear-gradient(135deg, #1a1a2e, #16213e); color: #eee; min-height: 100vh; margin: 0; padding: 40px 20px; }
    .container { max-width: 1000px; margin: 0 auto; }
    h1 { font-size: 3em; text-align: center; }
    .cta-btn { display: block; width: fit-content; margin: 0 auto; background: #64b5f6; color: #1a1a2e; padding: 16px 40px; font-size: 1.2em; font-weight: bold; border-radius: 8px; text-decoration: none; }
    .features { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 30px; margin: 60px 0; }
    .feature { background: rgba(255,255,255,0.05); padding: 30px; border-radius: 12px; }
  </style>
</head>
<body>
  <div class="container">
    <h1>[Name]</h1>
    <a href="[STORE_URL]" class="cta-btn">Add to Chrome</a>
    <div class="features">
      <div class="feature"><h3>Feature 1</h3><p>Description</p></div>
    </div>
  </div>
</body>
</html>
```

---

## Full Project Workflow Phases

### Phase 1: Planning & Setup
- Define extension purpose and features
- Decide on monetization (free / freemium / paid)
- Create project folder: `C:\Users\Carlos Faviel Font\[PROJECT_NAME]`
- Initialize npm: `npm init -y && npm install canvas playwright`
- Create manifest.json (MV3, permissions, host_permissions)

### Phase 2: Core Development
- Build content scripts, popup UI, background worker
- Implement core functionality
- Add chrome.storage for settings persistence

### Phase 3: Testing with Playwright
- Create test-full.js with test cases
- Run tests and fix errors
- Verify extension loads and features work

### Phase 4: Pro/Payment Features & Supporters Monetization

**Standard monetization tiers for ALL extensions:**
1. **Free** - Full features with non-intrusive ads
2. **Remove Ads ($4.99)** - One-time, no ads
3. **Supporter ($32)** - Lifetime, badge, name in credits
4. **Broadcast ($32)** - Custom message shown to ALL users forever

---

#### 4.1 Create Stripe Products

```bash
# Remove Ads tier
curl -s https://api.stripe.com/v1/products -u "$STRIPE_SECRET_KEY:" -d "name=[NAME] - Remove Ads"
curl -s https://api.stripe.com/v1/prices -u "$STRIPE_SECRET_KEY:" -d "product=prod_XXX" -d "unit_amount=499" -d "currency=usd"
curl -s https://api.stripe.com/v1/payment_links -u "$STRIPE_SECRET_KEY:" \
  -d "line_items[0][price]=price_XXX" -d "line_items[0][quantity]=1" \
  -d "metadata[product_type]=pro"

# Supporter tier
curl -s https://api.stripe.com/v1/products -u "$STRIPE_SECRET_KEY:" -d "name=[NAME] - Supporter"
curl -s https://api.stripe.com/v1/prices -u "$STRIPE_SECRET_KEY:" -d "product=prod_XXX" -d "unit_amount=3200" -d "currency=usd"
curl -s https://api.stripe.com/v1/payment_links -u "$STRIPE_SECRET_KEY:" \
  -d "line_items[0][price]=price_XXX" -d "line_items[0][quantity]=1" \
  -d "metadata[product_type]=lifetime"

# Broadcast tier (includes supporter benefits)
curl -s https://api.stripe.com/v1/products -u "$STRIPE_SECRET_KEY:" -d "name=[NAME] - Broadcast Message"
curl -s https://api.stripe.com/v1/prices -u "$STRIPE_SECRET_KEY:" -d "product=prod_XXX" -d "unit_amount=3200" -d "currency=usd"
# Note: Broadcast needs custom checkout with message field - set up in Stripe Dashboard
```

---

#### 4.2 Cloudflare Worker for License Validation & Supporters API

Create `worker/wrangler.toml`:
```toml
name = "[project]-api"
main = "src/index.js"
compatibility_date = "2024-01-01"

[vars]
STRIPE_WEBHOOK_SECRET = "" # Set via: wrangler secret put STRIPE_WEBHOOK_SECRET

[[kv_namespaces]]
binding = "LICENSES"
id = "" # Create via: wrangler kv namespace create LICENSES
```

Create `worker/src/index.js`:
```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const corsHeaders = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };

    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }

    if (url.pathname === '/validate' && request.method === 'GET') {
      return handleValidate(url, env, corsHeaders);
    }
    if (url.pathname === '/webhook' && request.method === 'POST') {
      return handleWebhook(request, env, corsHeaders);
    }
    if (url.pathname === '/supporters' && request.method === 'GET') {
      return handleGetSupporters(env, corsHeaders);
    }
    if (url.pathname === '/broadcasts' && request.method === 'GET') {
      return handleGetBroadcasts(env, corsHeaders);
    }

    return new Response('Not found', { status: 404, headers: corsHeaders });
  },
};

// License validation
async function handleValidate(url, env, headers) {
  const key = url.searchParams.get('key');
  if (!key) return json({ valid: false, error: 'Missing key' }, 400, headers);

  const record = await env.LICENSES.get(key.toUpperCase(), 'json');
  if (record && record.active !== false) {
    return json({ valid: true, tier: record.tier || 'pro' }, 200, headers);
  }
  return json({ valid: false }, 200, headers);
}

// Stripe webhook handler
async function handleWebhook(request, env, headers) {
  const body = await request.text();
  const sig = request.headers.get('stripe-signature');

  if (!verifySignature(body, sig, env.STRIPE_WEBHOOK_SECRET)) {
    return json({ error: 'Invalid signature' }, 401, headers);
  }

  const event = JSON.parse(body);

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object;
    const email = session.customer_email || session.customer_details?.email || '';
    const name = session.customer_details?.name || email.split('@')[0] || 'Anonymous';
    const productType = session.metadata?.product_type || 'pro';
    const broadcastMessage = session.metadata?.broadcast_message || '';
    const broadcastUrl = session.metadata?.broadcast_url || '';

    // Generate license key
    const licenseKey = generateKey();
    const tier = productType === 'broadcast' ? 'lifetime' : productType;

    // Save license
    await env.LICENSES.put(licenseKey, JSON.stringify({
      key: licenseKey, email, name, tier, productType,
      created: Date.now(), active: true,
      customerId: session.customer, sessionId: session.id,
    }));

    // Email → keys lookup
    const existing = await env.LICENSES.get(`email:${email}`, 'json') || [];
    existing.push(licenseKey);
    await env.LICENSES.put(`email:${email}`, JSON.stringify(existing));

    // Add to supporters (for lifetime and broadcast)
    if (productType === 'lifetime' || productType === 'broadcast') {
      await addSupporter(env, { name, email, date: Date.now(), licenseKey });
    }

    // Add broadcast message
    if (productType === 'broadcast' && broadcastMessage) {
      await addBroadcast(env, {
        name, email, message: broadcastMessage, url: broadcastUrl,
        date: Date.now(), active: true, licenseKey,
      });
    }

    console.log(`New ${productType} purchase: ${email} → ${licenseKey}`);
  }

  return json({ received: true }, 200, headers);
}

// Supporters list with numbered order
async function addSupporter(env, supporter) {
  const list = await env.LICENSES.get('supporters:list', 'json') || [];
  const counter = await env.LICENSES.get('supporters:counter', 'json') || { count: 0 };
  counter.count += 1;

  list.push({ number: counter.count, name: supporter.name, date: supporter.date });

  await env.LICENSES.put('supporters:list', JSON.stringify(list));
  await env.LICENSES.put('supporters:counter', JSON.stringify(counter));
  return counter.count;
}

// Broadcast messages with numbered order
async function addBroadcast(env, broadcast) {
  const list = await env.LICENSES.get('broadcasts:list', 'json') || [];
  const counter = await env.LICENSES.get('supporters:counter', 'json') || { count: 0 };
  counter.count += 1;

  list.push({
    number: counter.count, name: broadcast.name, message: broadcast.message,
    url: broadcast.url, date: broadcast.date, active: true, id: crypto.randomUUID(),
  });

  await env.LICENSES.put('broadcasts:list', JSON.stringify(list));
  await env.LICENSES.put('supporters:counter', JSON.stringify(counter));
  return counter.count;
}

async function handleGetSupporters(env, headers) {
  const list = await env.LICENSES.get('supporters:list', 'json') || [];
  return json({ supporters: list }, 200, headers);
}

async function handleGetBroadcasts(env, headers) {
  const list = await env.LICENSES.get('broadcasts:list', 'json') || [];
  const active = list.filter(b => b.active);
  return json({ broadcasts: active }, 200, headers);
}

// Helpers
function generateKey() {
  const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
  const parts = [];
  for (let p = 0; p < 4; p++) {
    let seg = '';
    for (let i = 0; i < 5; i++) seg += chars[Math.floor(Math.random() * chars.length)];
    parts.push(seg);
  }
  return parts.join('-');
}

async function verifySignature(payload, sigHeader, secret) {
  if (!sigHeader || !secret) return false;
  const parts = Object.fromEntries(sigHeader.split(',').map(p => p.trim().split('=')));
  const timestamp = parts.t, signature = parts.v1;
  if (!timestamp || !signature) return false;

  const enc = new TextEncoder();
  const key = await crypto.subtle.importKey('raw', enc.encode(secret), { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']);
  const signed = await crypto.subtle.sign('HMAC', key, enc.encode(`${timestamp}.${payload}`));
  const expected = Array.from(new Uint8Array(signed)).map(b => b.toString(16).padStart(2, '0')).join('');
  return expected === signature;
}

function json(data, status, headers) {
  return new Response(JSON.stringify(data), {
    status, headers: { ...headers, 'Content-Type': 'application/json' },
  });
}
```

**Deploy worker:**
```bash
cd worker
wrangler kv namespace create LICENSES  # Copy ID to wrangler.toml
wrangler secret put STRIPE_WEBHOOK_SECRET  # Paste from Stripe Dashboard
wrangler deploy
```

---

#### 4.3 Supporters Ticker CSS

Add to `popup.css`:
```css
/* Supporters credits - scrolling ticker */
.supporters-credits {
  flex: 1;
  min-width: 0;
  background: linear-gradient(180deg, #fffbeb, #fef3c7);
  border-left: 1px solid #fde68a;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}
.credits-header {
  font-size: var(--credit-name-font, 9px);
  font-weight: 600;
  color: #92400e;
  text-align: center;
  padding: 6px 4px;
  background: rgba(255,255,255,0.8);
  border-bottom: 1px solid #fde68a;
  text-transform: uppercase;
}
.credits-viewport {
  flex: 1;
  overflow: hidden;
  position: relative;
  mask-image: linear-gradient(180deg, transparent, black 10%, black 90%, transparent);
  -webkit-mask-image: linear-gradient(180deg, transparent, black 10%, black 90%, transparent);
}
.credits-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 14px;
  padding: 20px 8px;
  animation: credits-scroll 45s linear infinite;
}
.credits-content:hover { animation-play-state: paused; }
@keyframes credits-scroll {
  0% { transform: translateY(0); }
  100% { transform: translateY(-50%); }
}
.credit-name {
  font-size: var(--credit-font, 10px);
  color: #78350f;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
}
.credit-number { font-size: var(--credit-name-font, 9px); color: #b45309; font-weight: 700; }
.credit-supporter { color: #78350f; word-break: break-word; }
.credit-name.broadcast {
  background: linear-gradient(135deg, #dbeafe, #e0e7ff);
  padding: 8px 10px;
  border-radius: 6px;
  color: #1e40af;
  max-width: 100%;
}
.credit-name.broadcast .broadcast-by { font-weight: 600; color: #1e3a8a; }
.credit-name.broadcast a { color: #2563eb; text-decoration: none; }
```

---

#### 4.4 Supporters Ticker JavaScript

Add to `popup.js`:
```javascript
const API_BASE = 'https://[project]-api.[account].workers.dev';
const SUPPORTERS_URL = `${API_BASE}/supporters`;
const BROADCASTS_URL = `${API_BASE}/broadcasts`;

let supportersList = [];
let broadcastsList = [];

async function loadSupporters() {
  try {
    const [supportersRes, broadcastsRes] = await Promise.all([
      fetch(SUPPORTERS_URL).then(r => r.json()).catch(() => ({ supporters: [] })),
      fetch(BROADCASTS_URL).then(r => r.json()).catch(() => ({ broadcasts: [] }))
    ]);

    supportersList = (supportersRes.supporters || []).map(s => ({
      type: 'supporter', number: s.number, name: s.name || 'Anonymous', date: s.date
    }));

    broadcastsList = (broadcastsRes.broadcasts || []).map(b => ({
      type: 'broadcast', number: b.number, name: b.name || 'Sponsor',
      message: b.message || '', url: b.url || null, date: b.date
    }));
  } catch {
    supportersList = [];
    broadcastsList = [];
  }
  renderSupportersTicker();
}

function renderSupportersTicker() {
  const all = [...supportersList, ...broadcastsList]
    .sort((a, b) => (a.number || 0) - (b.number || 0));

  if (all.length === 0) {
    supportersCredits?.classList.add('hidden');
    return;
  }
  supportersCredits?.classList.remove('hidden');

  // Duplicate for seamless infinite scroll
  const duplicated = [...all, ...all];

  if (!creditsContent) return;

  creditsContent.innerHTML = duplicated.map(item => {
    const num = item.number || '?';
    if (item.type === 'broadcast') {
      const content = item.url
        ? `<a href="${esc(item.url)}" target="_blank">${esc(item.message)}</a>`
        : esc(item.message);
      return `<div class="credit-name broadcast">
        <span class="credit-number">#${num}</span>
        <span class="broadcast-by">📢 ${esc(item.name)}</span>
        ${content}
      </div>`;
    } else {
      return `<div class="credit-name">
        <span class="credit-number">#${num}</span>
        <span class="credit-supporter">⭐ ${esc(item.name)}</span>
      </div>`;
    }
  }).join('');

  // Slow cinematic scroll
  const duration = Math.max(40, all.length * 4);
  creditsContent.style.animationDuration = `${duration}s`;

  // Random start position so different supporters show each time
  const randomStart = Math.random() * duration;
  creditsContent.style.animationDelay = `-${randomStart}s`;
}

// Call on init
loadSupporters();
```

---

#### 4.5 Supporters Ticker HTML

Add to `popup.html`:
```html
<div class="main-content-wrapper">
  <!-- Supporters credits column -->
  <div id="supportersCredits" class="supporters-credits">
    <div class="credits-header">❤️ Supporters</div>
    <div class="credits-viewport">
      <div id="creditsContent" class="credits-content"></div>
    </div>
  </div>

  <!-- Main content -->
  <div id="codeList" class="code-list"></div>
</div>
```

---

#### 4.6 Settings UI for Purchases

Add to settings in `popup.html`:
```html
<h2>Remove Ads</h2>

<!-- Remove Ads Tier -->
<div class="tier-card">
  <div class="tier-header">
    <span class="tier-name">Remove Ads</span>
    <span class="tier-price">$4.99</span>
  </div>
  <div class="tier-benefits">
    <div class="benefit">✓ No ads ever</div>
    <div class="benefit">✓ Clean interface</div>
  </div>
  <button id="buyProBtn" class="btn btn-primary btn-tier">Remove Ads</button>
</div>

<!-- Supporter Tier -->
<div class="tier-card featured">
  <div class="tier-badge">SUPPORTER</div>
  <div class="tier-header">
    <span class="tier-name">Lifetime</span>
    <span class="tier-price">$32</span>
  </div>
  <div class="tier-benefits">
    <div class="benefit">✓ No ads ever</div>
    <div class="benefit">✓ Supporter badge</div>
    <div class="benefit">✓ Name in credits</div>
  </div>
  <button id="buyLifetimeBtn" class="btn btn-primary btn-tier btn-featured">Become Supporter</button>
</div>

<!-- Broadcast -->
<div class="broadcast-section">
  <h2>📢 Broadcast Your Message</h2>
  <p class="broadcast-desc">Get your message seen by all users!</p>
  <button id="buyBroadcastBtn" class="btn btn-broadcast">Get Broadcast Slot — $32</button>
</div>

<!-- License activation -->
<div class="license-section">
  <p class="license-label">Already purchased? Enter your license key:</p>
  <div class="license-row">
    <input type="text" id="licenseKey" placeholder="XXXXX-XXXXX-XXXXX-XXXXX">
    <button id="activateBtn" class="btn btn-secondary">Activate</button>
  </div>
</div>
<div id="licenseStatus"></div>
```

### Phase 5: Asset Generation
- Run generate-icons.js, generate-promo.js, take-screenshots.js
- **⚠️ OPEN AND VERIFY each image - check text is not cut off!**

### Phase 6: Website Deployment
```bash
npx wrangler pages project create [PROJECT] --production-branch main
npx wrangler pages deploy . --project-name=[PROJECT]
```

### Phase 7: GitHub Repository
```bash
git init && git add -A
curl -s -X POST -H "Authorization: token $GITHUB_TOKEN" https://api.github.com/user/repos -d '{"name":"REPO","private":true}'
git remote add origin https://$GITHUB_TOKEN@github.com/USERNAME/REPO.git
git push -u origin main
```

### Phase 8: Chrome Web Store Upload

**Create extension zip:**
```bash
powershell Compress-Archive -Path manifest.json,*.js,*.html,*.css,icons,lib -DestinationPath [name].zip
```

**Create a `store-setup/` folder with ALL assets in one place:**
```
store-setup/
├── [name].zip                  # The extension package
├── icon128.png                 # Store icon (128x128)
├── promo-small-440x280.png     # Small promo tile
├── promo-marquee-1400x560.png  # Marquee promo tile
├── screenshot-1.png            # Screenshot (1280x800)
└── SUBMIT-TO-STORE.txt         # ALL form text ready to copy
```

**The SUBMIT-TO-STORE.txt MUST contain:**
1. ALL dropdown selections (Category, Language)
2. ALL text fields (Description, Single Purpose)
3. ALL URLs (Homepage, Support, Privacy Policy)
4. ALL checkboxes to check/uncheck
5. ALL permission justifications
6. ALL certification checkboxes

**Example SUBMIT-TO-STORE.txt structure:**
```
================================================================================
                    CHROME WEB STORE SUBMISSION - [NAME]
================================================================================

Go to: https://chrome.google.com/webstore/devconsole

================================================================================
                              STORE LISTING TAB
================================================================================

CATEGORY: [Productivity / Developer Tools / etc.]
LANGUAGE: English

DESCRIPTION:
---
[Full description text]
---

GRAPHIC ASSETS:
  - Store icon: icon128.png
  - Small promo: promo-small-440x280.png
  - Marquee promo: promo-marquee-1400x560.png
  - Screenshot: screenshot-1.png

HOMEPAGE URL: https://[project].pages.dev
SUPPORT URL: https://[project].pages.dev/support.html
MATURE CONTENT: OFF

================================================================================
                            PRIVACY PRACTICES TAB
================================================================================

SINGLE PURPOSE DESCRIPTION (copy this):
---
[Extension name] is a [what it does in one sentence]. It [main function],
[secondary function], and [tertiary function if applicable].
---

PERMISSION JUSTIFICATIONS (one per permission in manifest.json):

  storage:
  ---
  [Explain what you store: user preferences, data, settings, etc.]
  ---

  clipboardWrite:
  ---
  [Explain why: copies X to clipboard when user does Y]
  ---

  identity:
  ---
  [Explain why: enables Google Sign-In for cloud sync, optional feature]
  ---

  activeTab:
  ---
  [Explain why: accesses current tab to do X when user clicks extension]
  ---

  tabs:
  ---
  [Explain why: needs to read tab URLs to do X]
  ---

  [Add justification for EVERY permission in your manifest.json]

--------------------------------------------------------------------------------

REMOTE CODE:
  ○ Select: "No, I am not using remote code"

  (Only select Yes if you load JS/WASM from external URLs)

--------------------------------------------------------------------------------

DATA USAGE - What user data do you collect?

CHECK if your extension collects this type of data:

  ☐ Personally identifiable information
    (name, address, EMAIL ADDRESS, age, identification number)
    → CHECK if you use Google Sign-In or collect any user info

  ☐ Health information
    (heart rate, medical history, symptoms, diagnoses)
    → Usually NO for most extensions

  ☐ Financial and payment information
    (transactions, credit cards, financial statements)
    → Usually NO - Stripe handles payments externally

  ☐ Authentication information
    (passwords, CREDENTIALS, security questions, PINs)
    → CHECK if you store passwords, API keys, TOTP secrets, etc.

  ☐ Personal communications
    (emails, texts, chat messages)
    → CHECK if you read/store user messages

  ☐ Location
    (region, IP address, GPS coordinates)
    → Usually NO for most extensions

  ☐ Web history
    (list of pages visited, page titles, time of visit)
    → CHECK if you track browsing history

  ☐ User activity
    (clicks, mouse position, scroll, keystroke logging)
    → CHECK if you monitor user interactions

  ☐ Website content
    (text, images, sounds, videos, hyperlinks)
    → CHECK if you read/modify page content

--------------------------------------------------------------------------------

CERTIFICATIONS - CHECK ALL THREE (required):

  ☑️ I do not sell or transfer user data to third parties,
     outside of the approved use cases

  ☑️ I do not use or transfer user data for purposes that
     are unrelated to my item's single purpose

  ☑️ I do not use or transfer user data to determine
     creditworthiness or for lending purposes

--------------------------------------------------------------------------------

PRIVACY POLICY URL: https://[project].pages.dev/privacy-policy.html

================================================================================
                              FILES IN THIS FOLDER
================================================================================

  [name].zip                    - Extension package
  icon128.png                   - Store icon
  promo-small-440x280.png       - Small promo
  promo-marquee-1400x560.png    - Marquee promo
  screenshot-1.png              - Screenshot
  SUBMIT-TO-STORE.txt           - This file
================================================================================
```

- User uploads to chrome.google.com/webstore/devconsole
- **ALWAYS provide ALL form fields in chat AND in SUBMIT-TO-STORE.txt**
- **ALWAYS open the store-setup folder for the user at the end**

---

## Infrastructure & Services Reference

| Service | Use For |
|---------|---------|
| Cloudflare Pages | Landing pages, privacy policies |
| Cloudflare Workers | License validation APIs |
| Stripe | Payments |
| GitHub | Code repository |
| Firebase | Google OAuth + Firestore |
| Supabase | PostgreSQL + Auth |
| Render | Full backend APIs |

**Typical Stack:**
```
Extension (client)
  ├── Cloudflare Pages (landing page)
  ├── Cloudflare Worker (license API) → Stripe
  └── Firebase/Supabase (user data sync)
```

---

## Quick Commands

**Cloudflare Pages:**
```bash
npx wrangler pages deploy . --project-name=PROJECT
```

**GitHub Private Repo:**
```bash
curl -s -X POST -H "Authorization: token $GITHUB_TOKEN" https://api.github.com/user/repos -d '{"name":"REPO","private":true}'
```

**Extension Zip:**
```bash
powershell Compress-Archive -Path manifest.json,content.js,popup.html,popup.js,styles.css,icon48.png,icon128.png -DestinationPath extension.zip
```

---

## Manual Steps (User Required)

| Step | Why Manual |
|------|-----------|
| Chrome Web Store upload | Requires authenticated browser |
| Store form filling | No API available |
| OAuth consent screen | Requires GCP Console |
| Extension testing | User loads unpacked |
