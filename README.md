const puppeteer = require('puppeteer');
const fs = require('fs');
const path = require('path');

(async () => {
  const browser = await puppeteer.launch({ headless: true });
  const page = await browser.newPage();

  // 1. Login
  await page.goto('https://example-intranet.com/login');
  await page.type('#userId', 'YOUR_ID');
  await page.type('#password', 'YOUR_PASSWORD');
  await Promise.all([
    page.waitForNavigation(),
    page.click('#login-button'),
  ]);

  // 2. Navigate to search system
  await page.click('a[href*="電話帳"]');
  await page.waitForSelector('a[href*="メールアドレス検索システム"]');
  await page.click('a[href*="メールアドレス検索システム"]');
  await page.waitForSelector('button#proceed-to-search');
  await page.click('button#proceed-to-search');

  // 3. Perform search
  await page.waitForSelector('input[type="checkbox"][name="本務"]');
  await page.click('input[type="checkbox"][name="本務"]');
  // ... (check other boxes)
  await page.click('button#search');
  await page.waitForSelector('button#export-csv');
  await page.click('button#export-csv');

  // 4. Capture download
  const downloadPath = path.resolve(__dirname, '../storage/app/exports');
  await page._client.send('Page.setDownloadBehavior', {
    behavior: 'allow',
    downloadPath,
  });

  // Wait for download to complete (simplified)
  await new Promise(resolve => setTimeout(resolve, 10000));

  await browser.close();
})();
