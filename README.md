<?php

// Add these at the top if not already there
use GuzzleHttp\Client;
use GuzzleHttp\Cookie\CookieJar;
use GuzzleHttp\Exception\GuzzleException;
use Illuminate\Http\Request; // For API request
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Response; // For JSON response
use Symfony\Component\DomCrawler\Crawler;
use GuzzleHttp\Psr7\Uri;
use GuzzleHttp\Psr7\UriResolver;

class ScraperEmailDataController extends Controller
{
    private $cookieJar;
    private $client;
    private $username = '53439';
    private $password = 'daiD627';
    private $groupCompanyCode = 'U';

    private $boxFolderIdPhoneBookData = '322352598808';
    private $boxFolderIdOrgInfo = '322352598808';
    private $boxFolderIdOrgInfoDoboku = '322352598808';

    // Store the API key in .env for better security
    private $apiKey;

    // URLs
    private const LOGIN_URL = 'http://login.fc.obayashi.co.jp/sso/UI/Login';
    private const MANUAL_ADDR_MAN_URL = 'http://it-nw.isc.obayashi.co.jp/pick/manual/addr_man.htm';
    private const MANUAL_MOKUJI_URL = 'http://it-nw.isc.obayashi.co.jp/pick/manual/mokuji.htm';
    private const ADDR103_URL = 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR103.aspx';

    public function __construct()
    {
        $this->username = env('SCRAPER_USERNAME', '53439');
        $this->password = env('SCRAPER_PASSWORD', 'daiD627');
        $this->groupCompanyCode = env('SCRAPER_GROUP_COMPANY_CODE', 'U');
        $this->apiKey = env('SCRAPER_API_KEY', 'YOUR_SUPER_SECRET_API_KEY'); // Set this in your .env

        $this->boxFolderIdPhoneBookData = env('BOX_FOLDER_ID_PHONE_BOOK_DATA', '322352598808');
        $this->boxFolderIdOrgInfo = env('BOX_FOLDER_ID_ORG_INFO', '322352598808');
        $this->boxFolderIdOrgInfoDoboku = env('BOX_FOLDER_ID_ORG_INFO_DOBOKU', '322352598808');

        $this->cookieJar = new CookieJar();
        $this->client = new Client([
            'cookies' => $this->cookieJar,
            'allow_redirects' => true,
            'timeout' => 60, // Increased timeout for potentially long scraping tasks
            'connect_timeout' => 10,
            'headers' => [
                'User-Agent' => 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36',
                'Accept' => 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
                'Accept-Encoding' => 'gzip, deflate',
                'Accept-Language' => 'ja-JP,ja;q=0.9,en-US;q=0.8,en;q=0.7',
                'Cache-Control' => 'no-cache',
                'Connection' => 'keep-alive',
                // 'Origin' and 'Referer' will be set dynamically or contextually where needed.
                // Let Guzzle handle Origin for POSTs based on the request URL.
                // Referer is often specific to a step.
                'Pragma' => 'no-cache',
                'Upgrade-Insecure-Requests' => '1',
            ],
        ]);
    }

    // Existing methods (getLoginFormParams, getSearchFormParamsPhoneBook, etc.) remain unchanged
    // ... (keep all your getSearchFormParams... methods) ...
    private function getLoginFormParams(): array
    {
        return [
            'IDToken0-0' => $this->groupCompanyCode,
            'IDToken0' => $this->groupCompanyCode,
            'IDToken1' => $this->groupCompanyCode . $this->username,
            'IDToken2' => $this->password,
            'Login.Submit' => 'ログイン',
            'goto' => 'aHR0cDovL2ludHJhbG9naW4uZmMub2JheWFzaGkuY28uanAvY2dpLWJpbi9jbG9naW4uY2dpP2FjY2Vzcz1odHRwOi8vd3d3LmZjLm9iYXlhc2hpLmNvLmpwLw==',
            'gotoOnFail' => '',
            'SunQueryParamsString' => 'dHlwZT1vYmF5YXNoaQ==',
            'encoded' => 'true',
            'type' => 'obayashi',
            'gx_charset' => 'UTF-8',
            'IDButton' => 'ログイン',
        ];
    }

    private function getSearchFormParamsPhoneBook(): array
    {
        return [
            'chkTen_09' => '0', 'chkTen_11' => '0', 'chkTen_15' => '0', 'chkTen_10' => '0',
            'chkTen_26' => '0', 'chkTen_12' => '0', 'chkTen_13' => '0', 'chkTen_14' => '0',
            'chkTen_16' => '0', 'chkTen_17' => '0', 'chkTen_19' => '0', 'chkTen_27' => '0',
            'chkTen_31' => '0', 'chkTen_32' => '0', 'chkTen_46' => '0', 'chkTen_36' => '0',
            'chkTen_66' => '0', 'txtSosikiCode' => '', 'txtSosikiName' => '',
            'chkJG_001' => '0', 'chkJG_002' => '1', // 現場
            'chkKKubun_001' => '1', // 国内土木
            'chkKKubun_002' => '0', 'chkKKubun_003' => '1', // 国内建築
            'chkKKubun_004' => '0', 'txtKojinCode' => '', 'rdoKyoryoku' => '0', 'txtKyoryoku' => '',
            'chkHonKen_001' => '0', 'chkHonKen_002' => '0', 'chkHonKen_003' => '0', 'chkHonKen_004' => '0',
            'ddtSecRankS' => '', 'ddtSecRankE' => '', 'rdoYaku' => '0', 'txtYaku' => '',
            'rdoJuKubun' => '0', 'txtJuKubun' => '', 'chkSyoku_001' => '0', 'chkSyoku_002' => '0',
            'chkSyoku_003' => '0', 'chkSyoku_004' => '0', 'chkSyoku_005' => '0', 'chkSyoku_006' => '0',
            'hidKojinCode' => '1', 'hidHonken' => '1', 'hidKojinName' => '1', 'hidKojinKana' => '0',
            'hidRank' => '0', 'hidMailAddress' => '0', 'hidYaku' => '0', 'hidTenCode' => '1',
            'hidJukubun' => '0', 'hidTenMei' => '1', 'hidJukubunName' => '0', 'hidSosikiCode' => '1',
            'hidSyoku' => '1', 'hidSosikiMei' => '1', 'hidNaisen' => '0', 'hidJGKubun' => '1',
            'hidGaisen' => '0', 'hidKoujiKubun' => '1', 'hidWorksite' => '0',
        ];
    }

    private function getSearchFormParamsOrgInfo(): array
    {
        return [
            'chkTen_09' => '0', 'chkTen_11' => '0', 'chkTen_15' => '0', 'chkTen_10' => '0',
            'chkTen_26' => '0', 'chkTen_12' => '0', 'chkTen_13' => '0', 'chkTen_14' => '0',
            'chkTen_16' => '0', 'chkTen_17' => '0', 'chkTen_19' => '0', 'chkTen_27' => '0',
            'chkTen_31' => '0', 'chkTen_32' => '0', 'chkTen_46' => '0', 'chkTen_36' => '0',
            'chkTen_66' => '0', 'txtSosikiCode' => '', 'txtSosikiName' => '',
            'chkJG_001' => '0', 'chkJG_002' => '0', 'chkKKubun_001' => '0', 'chkKKubun_002' => '0',
            'chkKKubun_003' => '0', 'chkKKubun_004' => '0', 'txtKojinCode' => '',
            'rdoKyoryoku' => '0', 'txtKyoryoku' => '', 'chkHonKen_001' => '0', 'chkHonKen_002' => '0',
            'chkHonKen_003' => '0', 'chkHonKen_004' => '0', 'ddtSecRankS' => '', 'ddtSecRankE' => '',
            'rdoYaku' => '0', 'txtYaku' => '', 'rdoJuKubun' => '0', 'txtJuKubun' => '',
            'chkSyoku_001' => '0', 'chkSyoku_002' => '0', 'chkSyoku_003' => '0', 'chkSyoku_004' => '0',
            'chkSyoku_005' => '0', 'chkSyoku_006' => '0', 'hidKojinCode' => '0', 'hidHonken' => '0',
            'hidKojinName' => '0', 'hidKojinKana' => '0', 'hidRank' => '0', 'hidMailAddress' => '0',
            'hidYaku' => '0', 'hidTenCode' => '1', 'hidJukubun' => '0', 'hidTenMei' => '1',
            'hidJukubunName' => '0', 'hidSosikiCode' => '1', 'hidSyoku' => '0', 'hidSosikiMei' => '1',
            'hidNaisen' => '0', 'hidJGKubun' => '1', 'hidGaisen' => '0', 'hidKoujiKubun' => '1',
            'hidWorksite' => '0',
        ];
    }

    private function getSearchFormParamsOrgInfoDoboku(): array
    {
        return [
            'chkTen_09' => '0', 'chkTen_11' => '0', 'chkTen_15' => '0', 'chkTen_10' => '0',
            'chkTen_26' => '0', 'chkTen_12' => '0', 'chkTen_13' => '0', 'chkTen_14' => '0',
            'chkTen_16' => '0', 'chkTen_17' => '0', 'chkTen_19' => '0', 'chkTen_27' => '0',
            'chkTen_31' => '0', 'chkTen_32' => '0', 'chkTen_46' => '0', 'chkTen_36' => '0',
            'chkTen_66' => '0', 'txtSosikiCode' => '', 'txtSosikiName' => '',
            'chkJG_001' => '0', 'chkJG_002' => '0', 'chkKKubun_001' => '0', 'chkKKubun_002' => '0',
            'chkKKubun_003' => '0', 'chkKKubun_004' => '0', 'txtKojinCode' => '',
            'rdoKyoryoku' => '0', 'txtKyoryoku' => '', 'chkHonKen_001' => '1', // 本社
            'chkHonKen_002' => '0', 'chkHonKen_003' => '0', 'chkHonKen_004' => '0',
            'ddtSecRankS' => '', 'ddtSecRankE' => '', 'rdoYaku' => '0', 'txtYaku' => '',
            'rdoJuKubun' => '0', 'txtJuKubun' => '', 'chkSyoku_001' => '0',
            'chkSyoku_002' => '1', // 土木
            'chkSyoku_003' => '0', 'chkSyoku_004' => '0', 'chkSyoku_005' => '0', 'chkSyoku_006' => '0',
            'hidKojinCode' => '1', 'hidHonken' => '1', 'hidKojinName' => '1', 'hidKojinKana' => '0',
            'hidRank' => '0', 'hidMailAddress' => '0', 'hidYaku' => '0', 'hidTenCode' => '1',
            'hidJukubun' => '0', 'hidTenMei' => '1', 'hidJukubunName' => '0', 'hidSosikiCode' => '1',
            'hidSyoku' => '0', 'hidSosikiMei' => '1', 'hidNaisen' => '0', 'hidJGKubun' => '1',
            'hidGaisen' => '0', 'hidKoujiKubun' => '1', 'hidWorksite' => '0',
        ];
    }

    /**
     * API endpoint for Power Automate to fetch CSV data.
     */
    public function getScrapedCsvData(Request $request)
    {
        // API Key Authentication
        if ($request->header('X-API-KEY') !== $this->apiKey) {
            Log::warning('API call with invalid API Key.');
            return Response::json(['error' => 'Unauthorized'], 401);
        }

        $reportType = $request->input('report_type');
        if (empty($reportType)) {
            return Response::json(['error' => 'Report type not specified'], 400);
        }

        Log::info("API call received for report type: {$reportType}");

        try {
            $this->login(); // Login for each API call to ensure session is fresh
            $this->callFinalUrl(); // Navigate to the state before search forms

            $csvData = null;
            switch ($reportType) {
                case 'phone_book':
                    $csvData = $this->fetchPhoneBookData();
                    break;
                case 'org_info':
                    $csvData = $this->fetchOrgInfoData();
                    break;
                case 'org_info_doboku':
                    $csvData = $this->fetchOrgInfoDobokuData();
                    break;
                default:
                    return Response::json(['error' => 'Invalid report type'], 400);
            }

            if ($csvData) {
                Log::info("Successfully fetched data for {$reportType}. Filename: {$csvData['fileName']}");
                // Power Automate expects the file content to be just the string.
                // If it needs to be base64 encoded for transfer, do it here:
                // 'csvData' => base64_encode($csvData['csvData'])
                return Response::json($csvData, 200);
            } else {
                Log::error("No CSV data returned for report type: {$reportType}");
                return Response::json(['error' => 'Failed to retrieve CSV data'], 500);
            }

        } catch (\Exception $e) {
            Log::error("API Error for report type {$reportType}: " . $e->getMessage(), ['exception' => $e]);
            return Response::json(['error' => 'Scraping process failed: ' . $e->getMessage()], 500);
        }
    }


    public function login(): void
    {
        try {
            // Set Origin for login
            $this->client->getConfig('headers')['Origin'] = 'http://login.fc.obayashi.co.jp';
            // Referer for login is often the login page itself or not strictly required if navigating directly
            unset($this->client->getConfig('headers')['Referer']);


            $response = $this->client->get(self::LOGIN_URL);
            Log::info('Accessed login page. Status: ' . $response->getStatusCode());

            $loginData = $this->getLoginFormParams();
            $response = $this->client->post(self::LOGIN_URL, [
                'form_params' => $loginData,
                'headers' => [ // Override Origin for this specific POST
                    'Origin' => 'http://login.fc.obayashi.co.jp',
                    'Referer' => self::LOGIN_URL // Referer is the login page
                ]
            ]);

            if ($response->getStatusCode() < 200 || $response->getStatusCode() >= 300) {
                $loginHtml = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'SJIS-win');
                Log::error('Login failed. Status: ' . $response->getStatusCode(), ['body' => $loginHtml]);
                throw new \Exception('Login failed. Status Code: ' . $response->getStatusCode());
            }
            Log::info('Successfully logged in.');
        } catch (GuzzleException $e) {
            Log::error('Guzzle error during login: ' . $e->getMessage(), ['exception' => $e]);
            throw new \Exception('An error occurred during login: ' . $e->getMessage(), 0, $e);
        }
    }

    private function callFinalUrl(): void
    {
        try {
            // Set Referer for these steps
            $this->client->getConfig('headers')['Referer'] = self::LOGIN_URL; // Or the page after login
            $this->client->getConfig('headers')['Origin'] = 'http://it-nw.isc.obayashi.co.jp';


            $this->client->get(self::MANUAL_ADDR_MAN_URL, ['headers' => ['Referer' => 'http://www.fc.obayashi.co.jp/']]); // Typical referer after login
            Log::info('Accessed MANUAL_ADDR_MAN_URL.');

            $mokujiResponse = $this->client->get(self::MANUAL_MOKUJI_URL, ['headers' => ['Referer' => self::MANUAL_ADDR_MAN_URL]]);
            Log::info('Accessed MANUAL_MOKUJI_URL.');

            $mokujiHtml = mb_convert_encoding((string) $mokujiResponse->getBody(), 'UTF-8', 'shift_jis');
            $mokujiCrawler = new Crawler($mokujiHtml);
            $formNode = $mokujiCrawler->filter('form[name="f"]');

            if ($formNode->count() === 0) {
                Log::error('Form "f" not found on mokuji.htm page.');
                throw new \Exception('Form "f" not found on mokuji.htm page.');
            }

            $formAction = $formNode->attr('action'); // e.g., ../frm/ADDR101.aspx
            $prevValue = $formNode->filter('input[name="prev"]')->attr('value'); // e.g., ADDR101
            
            $baseUri = new Uri(self::MANUAL_MOKUJI_URL);
            $absoluteNextPageUrl = (string) UriResolver::resolve($baseUri, new Uri($formAction));
            Log::info("Resolved next page URL: {$absoluteNextPageUrl} with prev: {$prevValue}");

            // The POST to ADDR101.aspx
            $response = $this->client->post($absoluteNextPageUrl, [
                'form_params' => ['prev' => $prevValue],
                'headers' => [
                    'Referer' => self::MANUAL_MOKUJI_URL,
                    'Origin' => (new Uri($absoluteNextPageUrl))->getScheme() . '://' . (new Uri($absoluteNextPageUrl))->getHost()
                ]
            ]);
            Log::info("POSTed to {$absoluteNextPageUrl}. Status: " . $response->getStatusCode());
            // After this POST, we are likely on ADDR101.aspx, ready to submit to ADDR103.aspx.
            // The default Referer for Guzzle client for subsequent ADDR103 posts will be $absoluteNextPageUrl.
            $this->client->getConfig('headers')['Referer'] = $absoluteNextPageUrl;


        } catch (GuzzleException $e) {
            Log::error("Guzzle error accessing manual pages: " . $e->getMessage(), ['exception' => $e]);
            throw new \Exception("Error accessing manual pages: " . $e->getMessage(), 0, $e);
        } catch (\Exception $e) {
            Log::error("Error in callFinalUrl: " . $e->getMessage(), ['exception' => $e]);
            throw $e;
        }
    }

    private function fetchPhoneBookData(): ?array
    {
        try {
            // ADDR103.aspx is typically posted to from ADDR101.aspx
            // The Referer should be the URL of ADDR101.aspx after the previous POST in callFinalUrl.
            // Guzzle's 'allow_redirects' usually handles setting the Referer correctly after a POST if it redirects.
            // If not, ensure $this->client->getConfig('headers')['Referer'] is correctly set from callFinalUrl.
            $this->client->getConfig('headers')['Origin'] = 'http://it-nw.isc.obayashi.co.jp';
            // $this->client->getConfig('headers')['Referer'] should be ADDR101.aspx
            
            $response = $this->client->post(self::ADDR103_URL, [
                'form_params' => $this->getSearchFormParamsPhoneBook()
            ]);
            Log::info('Submitted PhoneBook search to ADDR103.aspx. Status: ' . $response->getStatusCode());
            return $this->handleCsvDownload($response, $this->boxFolderIdPhoneBookData, '電話帳Data_');
        } catch (GuzzleException $e) {
            Log::error("Guzzle error during phone book CSV search: " . $e->getMessage(), ['exception' => $e]);
            throw new \Exception("Error during phone book CSV search: " . $e->getMessage(), 0, $e);
        }
    }

    private function fetchOrgInfoData(): ?array
    {
        try {
            $this->client->getConfig('headers')['Origin'] = 'http://it-nw.isc.obayashi.co.jp';
            // $this->client->getConfig('headers')['Referer'] should be ADDR101.aspx

            $response = $this->client->post(self::ADDR103_URL, [
                'form_params' => $this->getSearchFormParamsOrgInfo()
            ]);
            Log::info('Submitted OrgInfo search to ADDR103.aspx. Status: ' . $response->getStatusCode());
            return $this->handleCsvDownload($response, $this->boxFolderIdOrgInfo, '組織情報_');
        } catch (GuzzleException $e) {
            Log::error("Guzzle error during org info CSV (type 1) search: " . $e->getMessage(), ['exception' => $e]);
            throw new \Exception("Error during organization info CSV (type 1) search: " . $e->getMessage(), 0, $e);
        }
    }

    private function fetchOrgInfoDobokuData(): ?array
    {
        try {
            $this->client->getConfig('headers')['Origin'] = 'http://it-nw.isc.obayashi.co.jp';
            // $this->client->getConfig('headers')['Referer'] should be ADDR101.aspx

            $response = $this->client->post(self::ADDR103_URL, [
                'form_params' => $this->getSearchFormParamsOrgInfoDoboku()
            ]);
            Log::info('Submitted OrgInfoDoboku search to ADDR103.aspx. Status: ' . $response->getStatusCode());
            return $this->handleCsvDownload($response, $this->boxFolderIdOrgInfoDoboku, '組織情報＿土木_');
        } catch (GuzzleException $e) {
            Log::error("Guzzle error during org info CSV (type 2) search: " . $e->getMessage(), ['exception' => $e]);
            throw new \Exception("Error during organization info CSV (type 2) search: " . $e->getMessage(), 0, $e);
        }
    }

    private function handleCsvDownload($response, string $folderId, string $filePrefix): ?array
    {
        if ($response->getStatusCode() === 200) {
            $contentType = $response->getHeaderLine('Content-Type');
            Log::info("Handling CSV download. Content-Type: {$contentType}");

            if (strpos($contentType, 'text/csv') !== false || strpos($contentType, 'application/octet-stream') !== false || strpos($contentType, 'application/vnd.ms-excel') !== false) {
                $csvData = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'shift_jis');
                $fileName = $filePrefix . date('Ymd_His') . '.csv'; // Added His for uniqueness if run multiple times a day

                // Optional: Save locally as a backup
                // $localPath = storage_path('app/scraped_csvs/'); // Example path
                // if (!is_dir($localPath)) {
                //     mkdir($localPath, 0755, true);
                // }
                // file_put_contents($localPath . $fileName, $csvData);
                // Log::info("CSV saved locally: " . $localPath . $fileName);

                return [
                    'fileName' => $fileName,
                    'csvData' => $csvData, // Raw CSV string
                    'boxFolderId' => $folderId
                ];
            } else {
                $bodyContent = (string) $response->getBody();
                Log::error("Expected CSV data, but got Content-Type: {$contentType}", ['body_preview' => substr($bodyContent, 0, 500)]);
                throw new \Exception("Expected CSV data in response, but got: " . $contentType . ". Check logs for body preview.");
            }
        } else {
            Log::error("Error submitting search that should yield CSV. Status: " . $response->getStatusCode(), ['body' => (string) $response->getBody()]);
            throw new \Exception("Error on page that should provide CSV. Status: " . $response->getStatusCode());
        }
        return null;
    }
}
