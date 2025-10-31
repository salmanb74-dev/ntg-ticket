/*
  Screenshot capture for NTG-Ticket staging.
  Usage (PowerShell):
  $env:STAGING_URL="https://your-staging-host/"; $env:ADMIN_EMAIL="admin@example.com"; $env:ADMIN_PASSWORD="password"; npm run capture:screens

  Outputs PNGs into docs/screens/ with filenames used by NTG-Ticket-User-Guide.md
*/

const fs = require('fs');
const path = require('path');
const puppeteer = require('puppeteer');

async function ensureDir(dirPath) {
  if (!fs.existsSync(dirPath)) {
    fs.mkdirSync(dirPath, { recursive: true });
  }
}

async function clickButtonByText(page, texts) {
  try {
    const handles = await page.$$('a,button');
    for (const h of handles) {
      const txt = (await page.evaluate(el => el.textContent || '', h)).trim().toLowerCase();
      if (texts.some(t => txt.includes(t))) {
        await h.click();
        return true;
      }
    }
  } catch (_) {}
  return false;
}

async function bypassNgrokInterstitial(page) {
  try {
    // Quick content sniff
    const content = (await page.content()).toLowerCase();
    const looksLikeNgrokWarning = content.includes('you should only visit sites you trust') || content.includes('visit site');
    if (looksLikeNgrokWarning) {
      // Try obvious controls
      const clickedKnown = await page.$('#visit-site')
        .then(async el => { if (el) { await el.click(); return true; } return false; })
        .catch(() => false);
      if (!clickedKnown) {
        const clickedText = await clickButtonByText(page, ['visit site', 'visit website', 'continue', 'proceed']);
        if (!clickedText) {
          // As a last resort, click the first primary-looking button
          const anyButton = await page.$('a,button');
          if (anyButton) await anyButton.click();
        }
      }
      await new Promise(r => setTimeout(r, 1200));
    }
  } catch (_) {}
}

async function saveShot(page, filePath, opts = {}) {
  await page.screenshot({ path: filePath, fullPage: false, ...opts });
  // eslint-disable-next-line no-console
  console.log(`Saved: ${filePath}`);
}

async function login(page, baseUrl, email, password) {
  // Navigate to root; if it redirects to login, handle it. Otherwise go explicitly.
  await page.goto(baseUrl, { waitUntil: 'networkidle2' });

  // Try common auth paths
  const candidatePaths = ['auth/signin', 'login', 'auth/login'];
  let atLogin = false;
  for (const p of candidatePaths) {
    if (page.url().includes(p)) {
      atLogin = true;
      break;
    }
  }
  if (!atLogin) {
    for (const p of candidatePaths) {
      await page.goto(new URL(p, baseUrl).toString(), { waitUntil: 'networkidle2' });
      if (page.url().includes(p)) {
        atLogin = true;
        break;
      }
    }
  }

  // Attempt a generic login form fill
  // These selectors may need adjusting depending on the staging UI
  // Try common input names and placeholders
  const emailSelectors = ['input[name="email"]', 'input[type="email"]', 'input[autocomplete="email"]'];
  const passwordSelectors = ['input[name="password"]', 'input[type="password"]', 'input[autocomplete="current-password"]'];
  const loginButtonSelectors = ['button[type="submit"]', 'button[name="submit"]', 'form button'];

  // Fill email
  for (const sel of emailSelectors) {
    const el = await page.$(sel);
    if (el) {
      await el.click({ clickCount: 3 });
      await el.type(email);
      break;
    }
  }
  // Fill password
  for (const sel of passwordSelectors) {
    const el = await page.$(sel);
    if (el) {
      await el.click({ clickCount: 3 });
      await el.type(password);
      break;
    }
  }
  // Click submit
  let clicked = false;
  for (const sel of loginButtonSelectors) {
    const el = await page.$(sel);
    if (el) {
      await el.click();
      clicked = true;
      break;
    }
  }
  if (!clicked) {
    // Fallback: try submitting the form by Enter
    await page.keyboard.press('Enter');
  }

  // Do not block on navigation; some auth flows don't trigger a full nav
  await new Promise((resolve) => setTimeout(resolve, 2000));
}

async function capture() {
  const baseUrl = process.env.STAGING_URL;
  const adminEmail = process.env.ADMIN_EMAIL;
  const adminPassword = process.env.ADMIN_PASSWORD;

  if (!baseUrl || !adminEmail || !adminPassword) {
    throw new Error('Missing environment variables. Required: STAGING_URL, ADMIN_EMAIL, ADMIN_PASSWORD');
  }

  const outDir = path.resolve(process.cwd(), 'docs', 'screens');
  await ensureDir(outDir);

  const browser = await puppeteer.launch({
    headless: 'new',
    defaultViewport: { width: 1440, height: 900 },
    ignoreHTTPSErrors: true,
    args: [
      '--ignore-certificate-errors',
      '--ignore-certificate-errors-spki-list',
    ],
  });
  const page = await browser.newPage();
  await page.setDefaultNavigationTimeout(60000);
  await page.setDefaultTimeout(60000);

  try {
    // Attempt to bypass common Chrome/Ngrok interstitials if present
    try {
      await page.goto(baseUrl, { waitUntil: 'domcontentloaded' });
      const details = await page.$('#details-button');
      if (details) {
        await details.click();
        const proceed = await page.$('#proceed-link');
        if (proceed) {
          await proceed.click();
          await new Promise(r => setTimeout(r, 800));
        }
      }
      await bypassNgrokInterstitial(page);
    } catch (_) {}

    await login(page, baseUrl, adminEmail, adminPassword);

    // 1) Create Ticket
    try {
      await page.goto(new URL('/tickets/create', baseUrl).toString(), { waitUntil: 'networkidle2' });
      await bypassNgrokInterstitial(page);
      // Try selecting a category to reveal custom fields
      const categorySelectors = ['select[name="category"]', 'select#category'];
      for (const sel of categorySelectors) {
        if (await page.$(sel)) {
          await page.select(sel, (await page.$eval(sel, el => el.options[1]?.value)) || '');
          break;
        }
      }
      await saveShot(page, path.join(outDir, 'create-ticket.png'));
    } catch (e) {
      console.warn('Create ticket capture failed:', e.message);
    }

    // 2) Ticket List
    try {
      await page.goto(new URL('/tickets', baseUrl).toString(), { waitUntil: 'networkidle2' });
      await bypassNgrokInterstitial(page);
      await saveShot(page, path.join(outDir, 'ticket-list.png'));
    } catch (e) {
      console.warn('Ticket list capture failed:', e.message);
    }

    // 3) Ticket Details with Comments (open first ticket if available)
    try {
      // Ensure we are on tickets page
      await page.goto(new URL('/tickets', baseUrl).toString(), { waitUntil: 'networkidle2' });
      await bypassNgrokInterstitial(page);
      const firstTicketLink = await page.$('a[href^="/tickets/"]');
      if (firstTicketLink) {
        await firstTicketLink.click();
  try {
    await page.waitForNavigation({ waitUntil: 'networkidle2', timeout: 60000 });
  } catch (_) {
    // If no navigation occurred, proceed; some auth flows update content without a full nav
  }
        await bypassNgrokInterstitial(page);
        await saveShot(page, path.join(outDir, 'ticket-comments.png'));
      }
    } catch (e) {
      console.warn('Ticket comments capture failed:', e.message);
    }

    // 4) Reports
    try {
      await page.goto(new URL('/reports', baseUrl).toString(), { waitUntil: 'networkidle2' });
      await bypassNgrokInterstitial(page);
      await saveShot(page, path.join(outDir, 'reports.png'));
    } catch (e) {
      console.warn('Reports capture failed:', e.message);
    }

    // 5) Theme Settings (Admin)
    try {
      await page.goto(new URL('/admin/theme-settings', baseUrl).toString(), { waitUntil: 'networkidle2' });
      await bypassNgrokInterstitial(page);
      await saveShot(page, path.join(outDir, 'theme-settings.png'));
    } catch (e) {
      console.warn('Theme settings capture failed:', e.message);
    }

    // 6) Custom Fields (try again on create page after category selection)
    try {
      await page.goto(new URL('/tickets/create', baseUrl).toString(), { waitUntil: 'networkidle2' });
      await bypassNgrokInterstitial(page);
      await saveShot(page, path.join(outDir, 'custom-fields.png'));
    } catch (e) {
      console.warn('Custom fields capture failed:', e.message);
    }

  } finally {
    await browser.close();
  }
}

capture().catch(err => {
  // eslint-disable-next-line no-console
  console.error(err);
  process.exit(1);
});


