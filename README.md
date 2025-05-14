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


<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Storage;
use Microsoft\Graph\Graph;
use Microsoft\Graph\Model;
use \GuzzleHttp\Client;

class ScrapeAndUpload extends Command
{
    protected $signature = 'scrape:emails';
    protected $description = 'Scrape data and upload to Box and Teams';

    public function handle()
    {
        $this->info('Running Puppeteer script...');
        exec('node scripts/scrape.js');

        $this->info('Finding latest CSV...');
        $files = Storage::files('exports');
        $latestFile = collect($files)->sortByDesc(fn($f) => Storage::lastModified($f))->first();

        if (!$latestFile) {
            $this->error('No file found.');
            return;
        }

        // Upload to Box
        $this->info('Uploading to Box...');
        $this->uploadToBox(storage_path("app/{$latestFile}"));

        // Upload to Teams
        $this->info('Uploading to Teams...');
        $this->uploadToTeams(storage_path("app/{$latestFile}"));

        $this->info('Done.');
    }

    protected function uploadToBox($filePath)
    {
        $accessToken = 'YOUR_BOX_ACCESS_TOKEN';
        $folderId = 'YOUR_FOLDER_ID';

        $client = new \GuzzleHttp\Client();
        $res = $client->request('POST', "https://upload.box.com/api/2.0/files/content", [
            'headers' => [
                'Authorization' => "Bearer $accessToken",
            ],
            'multipart' => [
                ['name' => 'attributes', 'contents' => json_encode(['name' => basename($filePath), 'parent' => ['id' => $folderId]])],
                ['name' => 'file', 'contents' => fopen($filePath, 'r')],
            ]
        ]);

        return json_decode($res->getBody(), true);
    }

    protected function uploadToTeams($filePath)
    {
        $accessToken = 'YOUR_GRAPH_API_TOKEN';
        $teamDriveId = 'YOUR_TEAM_DRIVE_ID';
        $folderPath = '/General/Documents';

        $client = new \GuzzleHttp\Client();
        $url = "https://graph.microsoft.com/v1.0/drives/{$teamDriveId}/root:{$folderPath}/".basename($filePath).":/content";

        $res = $client->put($url, [
            'headers' => [
                'Authorization' => "Bearer $accessToken",
                'Content-Type' => 'text/csv'
            ],
            'body' => fopen($filePath, 'r')
        ]);

        return json_decode($res->getBody(), true);
    }
}

