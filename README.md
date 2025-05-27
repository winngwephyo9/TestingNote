No, that is not possible and is fundamentally flawed for several critical reasons.

You're describing trying to automate a user's interaction with an OAuth 2.0 flow as if your scraper were a human user. While you could technically use a headless browser (like Puppeteer or Selenium) to navigate and click buttons, it will not yield a usable, long-term token for your server-side scraping application.

Here's why this approach fails:

The Box Login is an External Redirect:

When a user clicks a "Box Login" button on your website using OAuth 2.0 (Authorization Code Grant), they are redirected to Box's official login page (accounts.box.com).
Your scraper would then be interacting with Box's website, not your own.
After successful login and authorization on Box's site, Box then redirects the user back to a pre-registered redirect URI on your website. This redirect URI includes the code (authorization code).
Your server-side scraper running Guzzle cannot intercept this browser-based redirect. It doesn't have a browser context to receive that redirect from Box.
The Token Exchange is Server-Side for the code:

Even if your scraper could somehow get the code from Box's redirect, that code is then exchanged for an access_token and refresh_token by your server-side application making a direct POST request to Box's token endpoint, not by simulating clicks in a browser.
Security Measures and Bot Detection:

Box and other major services have sophisticated bot detection and security mechanisms. Trying to automate their login page directly will likely trigger CAPTCHAs, IP blocking, or other security challenges.
They can easily detect that an automated script is trying to mimic a human.
No Refresh Tokens from "Scraping":

Even in the extremely unlikely event you somehow managed to "scrape" an access_token (which you can't, as it's returned to your backend), you wouldn't get a refresh_token in a way that your server could persistently use. The whole point of refresh tokens is to keep your application authorized without repeated user interaction, but this is granted by Box to your registered application after the initial user consent, not by simply extracting it from a simulated browser session.
Violation of Terms of Service:

This method constitutes unauthorized access and bypassing intended security measures. It's a direct violation of Box's (and almost any service's) Terms of Service and could lead to your Box account being suspended or banned.
In summary: Your PHP Guzzle client cannot "scrape" a Box login button on your website to get an OAuth token because the OAuth flow involves browser redirects to Box's domain and then a server-side token exchange that the scraper cannot complete in a browser context.

The fundamental problem remains: You need Box admin authorization for any automated, server-to-server access (Client Credentials Grant, JWT) or for a standard OAuth 2.0 application to be considered "trusted" within an enterprise environment (which might be why the admin is blocking it).

What are your ONLY legitimate paths forward given your constraints (no admin approval for server-to-server apps):

Standard OAuth 2.0 (Authorization Code Grant) with manual initial user consent and refresh tokens:

This is the only way an individual user can grant your app access to their own Box account without admin authorization for the app itself (assuming the admin hasn't outright blocked all unpublished apps).
How it works:
Your "Box Login" button on your website would trigger the OAuth 2.0 redirect to accounts.box.com.
A human user (who has a Box account) clicks it, logs into Box, and explicitly authorizes your app.
Box redirects their browser back to your registered redirect URI.
Your backend (Laravel route) receives the code from this redirect.
Your backend immediately exchanges this code for an access_token and a refresh_token using Guzzle.
You store the refresh_token securely in your database, associated with that user.
When the access_token expires, your application uses the stored refresh_token to get a new access_token (and usually a new refresh_token). This allows your application to access that specific user's Box files for extended periods.
Limitation: This token is tied to a specific user. If you need to upload to a central folder or access other users' data, you'd need multiple users to go through this manual authorization process, or the central folder would have to be owned by the authorizing user.
Negotiate with the Admin (Still the Best Long-Term Solution): There might be a misunderstanding or a way to address their security concerns. Presenting a clear use case for a secure, server-to-server integration (like Client Credentials Grant) and explaining how it's more secure than manual processes or attempts to bypass security might change their mind.

You cannot bypass the core security model of Box's API. If an admin explicitly denies authorization for app types designed for server-to-server interaction, you are left with user-driven authorization, which inherently requires initial human interaction.



class ScraperEmailDataController extends Controller
{
    private $cookieJar;
    private $client;
    private $username = '53439'; //ユーザー名に
    private $password = 'daiD627'; //パスワードに
    private $groupCompanyCode = 'U'; //グループ会社コードに
    // Box Folder IDs
    private $boxFolderIdPhoneBookData = '322352598808';
    private $boxFolderIdOrgInfo = '322352598808';
    private $boxFolderIdOrgInfoDoboku = '322352598808';
    protected $url_path;
    protected $box_access_token;

    // URLs
    private const LOGIN_URL = 'http://login.fc.obayashi.co.jp/sso/UI/Login';
    private const MANUAL_ADDR_MAN_URL = 'http://it-nw.isc.obayashi.co.jp/pick/manual/addr_man.htm';
    private const MANUAL_MOKUJI_URL = 'http://it-nw.isc.obayashi.co.jp/pick/manual/mokuji.htm';
    private const ADDR103_URL = 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR103.aspx';

    public function __construct()
    {
        // It's better to get these from environment variables or a configuration file
        // For example: config('services.scraper.username'), config('services.scraper.password')
        $this->username = env('SCRAPER_USERNAME', '53439');
        $this->password = env('SCRAPER_PASSWORD', 'daiD627');
        $this->groupCompanyCode = env('SCRAPER_GROUP_COMPANY_CODE', 'U');

        $this->boxFolderIdPhoneBookData = env('BOX_FOLDER_ID_PHONE_BOOK_DATA', '322352598808');
        $this->boxFolderIdOrgInfo = env('BOX_FOLDER_ID_ORG_INFO', '322352598808');
        $this->boxFolderIdOrgInfoDoboku = env('BOX_FOLDER_ID_ORG_INFO_DOBOKU', '322352598808');


        $this->cookieJar = new CookieJar();
        $this->client = new Client([
            'cookies' => $this->cookieJar,
            'allow_redirects' => true,
            'headers' => [
                'User-Agent' => 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36',
                'Accept' => 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
                'Accept-Encoding' => 'gzip, deflate',
                'Accept-Language' => 'ja-JP,ja;q=0.9,en-US;q=0.8,en;q=0.7',
                'Cache-Control' => 'no-cache',
                'Connection' => 'keep-alive',
                'Origin' => 'http://login.fc.obayashi.co.jp',
                'Pragma' => 'no-cache',
                'Referer' => 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR101.aspx',
                'Upgrade-Insecure-Requests' => '1',
            ],
        ]);
    }

    public function scrapeAndStoreEmailData()
    {
        try {
            if (session()->has('url_path')) {
                $this->url_path = session('url_path');
                Log::info('url_path ' . $this->url_path);
            }
            $this->box_access_token = $this->getBoxAccessToken(); // Get token dynamically
            $this->login();
            $this->callFinalUrl();
            // All operations successful
            return 'CSV files downloaded and uploaded to Box successfully.';
        } catch (\Exception $e) {
            // Log the error for debugging
            log::error('Scraping and storage failed: ' . $e->getMessage(), ['exception' => $e]);
            return 'Error: ' . $e->getMessage();
        }
    }

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
            'chkTen_09' => '0',
            'chkTen_11' => '0',
            'chkTen_15' => '0',
            'chkTen_10' => '0',
            'chkTen_26' => '0',
            'chkTen_12' => '0',
            'chkTen_13' => '0',
            'chkTen_14' => '0',
            'chkTen_16' => '0',
            'chkTen_17' => '0',
            'chkTen_19' => '0',
            'chkTen_27' => '0',
            'chkTen_31' => '0',
            'chkTen_32' => '0',
            'chkTen_46' => '0',
            'chkTen_36' => '0',
            'chkTen_66' => '0',
            'txtSosikiCode' => '',
            'txtSosikiName' => '',
            'chkJG_001' => '0',
            'chkJG_002' => '1', // 現場
            'chkKKubun_001' => '1', // 国内土木
            'chkKKubun_002' => '0',
            'chkKKubun_003' => '1', // 国内建築
            'chkKKubun_004' => '0',
            'txtKojinCode' => '',
            'rdoKyoryoku' => '0',
            'txtKyoryoku' => '',
            'chkHonKen_001' => '0',
            'chkHonKen_002' => '0',
            'chkHonKen_003' => '0',
            'chkHonKen_004' => '0',
            'ddtSecRankS' => '',
            'ddtSecRankE' => '',
            'rdoYaku' => '0',
            'txtYaku' => '',
            'rdoJuKubun' => '0',
            'txtJuKubun' => '',
            'chkSyoku_001' => '0',
            'chkSyoku_002' => '0',
            'chkSyoku_003' => '0',
            'chkSyoku_004' => '0',
            'chkSyoku_005' => '0',
            'chkSyoku_006' => '0',
            'hidKojinCode' => '1',
            'hidHonken' => '1',
            'hidKojinName' => '1',
            'hidKojinKana' => '0',
            'hidRank' => '0',
            'hidMailAddress' => '0',
            'hidYaku' => '0',
            'hidTenCode' => '1',
            'hidJukubun' => '0',
            'hidTenMei' => '1',
            'hidJukubunName' => '0',
            'hidSosikiCode' => '1',
            'hidSyoku' => '1',
            'hidSosikiMei' => '1',
            'hidNaisen' => '0',
            'hidJGKubun' => '1',
            'hidGaisen' => '0',
            'hidKoujiKubun' => '1',
            'hidWorksite' => '0',
        ];
    }

    private function getSearchFormParamsOrgInfo(): array
    {
        return [
            'chkTen_09' => '0',
            'chkTen_11' => '0',
            'chkTen_15' => '0',
            'chkTen_10' => '0',
            'chkTen_26' => '0',
            'chkTen_12' => '0',
            'chkTen_13' => '0',
            'chkTen_14' => '0',
            'chkTen_16' => '0',
            'chkTen_17' => '0',
            'chkTen_19' => '0',
            'chkTen_27' => '0',
            'chkTen_31' => '0',
            'chkTen_32' => '0',
            'chkTen_46' => '0',
            'chkTen_36' => '0',
            'chkTen_66' => '0',
            'txtSosikiCode' => '',
            'txtSosikiName' => '',
            'chkJG_001' => '0',
            'chkJG_002' => '0',
            'chkKKubun_001' => '0',
            'chkKKubun_002' => '0',
            'chkKKubun_003' => '0',
            'chkKKubun_004' => '0',
            'txtKojinCode' => '',
            'rdoKyoryoku' => '0',
            'txtKyoryoku' => '',
            'chkHonKen_001' => '0',
            'chkHonKen_002' => '0',
            'chkHonKen_003' => '0',
            'chkHonKen_004' => '0',
            'ddtSecRankS' => '',
            'ddtSecRankE' => '',
            'rdoYaku' => '0',
            'txtYaku' => '',
            'rdoJuKubun' => '0',
            'txtJuKubun' => '',
            'chkSyoku_001' => '0',
            'chkSyoku_002' => '0',
            'chkSyoku_003' => '0',
            'chkSyoku_004' => '0',
            'chkSyoku_005' => '0',
            'chkSyoku_006' => '0',
            'hidKojinCode' => '0',
            'hidHonken' => '0',
            'hidKojinName' => '0',
            'hidKojinKana' => '0',
            'hidRank' => '0',
            'hidMailAddress' => '0',
            'hidYaku' => '0',
            'hidTenCode' => '1',
            'hidJukubun' => '0',
            'hidTenMei' => '1',
            'hidJukubunName' => '0',
            'hidSosikiCode' => '1',
            'hidSyoku' => '0',
            'hidSosikiMei' => '1',
            'hidNaisen' => '0',
            'hidJGKubun' => '1',
            'hidGaisen' => '0',
            'hidKoujiKubun' => '1',
            'hidWorksite' => '0',
        ];
    }

    private function getSearchFormParamsOrgInfoDoboku(): array
    {
        return [
            'chkTen_09' => '0',
            'chkTen_11' => '0',
            'chkTen_15' => '0',
            'chkTen_10' => '0',
            'chkTen_26' => '0',
            'chkTen_12' => '0',
            'chkTen_13' => '0',
            'chkTen_14' => '0',
            'chkTen_16' => '0',
            'chkTen_17' => '0',
            'chkTen_19' => '0',
            'chkTen_27' => '0',
            'chkTen_31' => '0',
            'chkTen_32' => '0',
            'chkTen_46' => '0',
            'chkTen_36' => '0',
            'chkTen_66' => '0',
            'txtSosikiCode' => '',
            'txtSosikiName' => '',
            'chkJG_001' => '0',
            'chkJG_002' => '0',
            'chkKKubun_001' => '0',
            'chkKKubun_002' => '0',
            'chkKKubun_003' => '0',
            'chkKKubun_004' => '0',
            'txtKojinCode' => '',
            'rdoKyoryoku' => '0',
            'txtKyoryoku' => '',
            'chkHonKen_001' => '1', // 本社 (assuming this is for doboku based on original code)
            'chkHonKen_002' => '0',
            'chkHonKen_003' => '0',
            'chkHonKen_004' => '0',
            'ddtSecRankS' => '',
            'ddtSecRankE' => '',
            'rdoYaku' => '0',
            'txtYaku' => '',
            'rdoJuKubun' => '0',
            'txtJuKubun' => '',
            'chkSyoku_001' => '0',
            'chkSyoku_002' => '1', // 土木 (assuming this is for doboku based on original code)
            'chkSyoku_003' => '0',
            'chkSyoku_004' => '0',
            'chkSyoku_005' => '0',
            'chkSyoku_006' => '0',
            'hidKojinCode' => '1',
            'hidHonken' => '1',
            'hidKojinName' => '1',
            'hidKojinKana' => '0',
            'hidRank' => '0',
            'hidMailAddress' => '0',
            'hidYaku' => '0',
            'hidTenCode' => '1',
            'hidJukubun' => '0',
            'hidTenMei' => '1',
            'hidJukubunName' => '0',
            'hidSosikiCode' => '1',
            'hidSyoku' => '0',
            'hidSosikiMei' => '1',
            'hidNaisen' => '0',
            'hidJGKubun' => '1',
            'hidGaisen' => '0',
            'hidKoujiKubun' => '1',
            'hidWorksite' => '0',
        ];
    }

    public function login(): void
    {
        try {
            $response = $this->client->get(self::LOGIN_URL);
            // Consider checking the initial response to ensure it's a login page

            $loginData = $this->getLoginFormParams();
            $response = $this->client->post(self::LOGIN_URL, ['form_params' => $loginData]);

            if ($response->getStatusCode() < 200 || $response->getStatusCode() >= 300) {
                $loginHtml = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'SJIS-win');
                throw new \Exception('Login failed. Status Code: ' . $response->getStatusCode() . ' Body: ' . $loginHtml);
            }
            log::info('Successfully logged in.');
        } catch (GuzzleException $e) {
            throw new \Exception('An error occurred during login: ' . $e->getMessage(), 0, $e);
        }
    }

    private function callFinalUrl(): void
    {
        try {
            $this->client->get(self::MANUAL_ADDR_MAN_URL);
            $mokujiResponse = $this->client->get(self::MANUAL_MOKUJI_URL);
            $mokujiHtml = mb_convert_encoding((string) $mokujiResponse->getBody(), 'UTF-8', 'shift_jis');
            $mokujiCrawler = new Crawler($mokujiHtml);
            $formNode = $mokujiCrawler->filter('form[name="f"]');

            if ($formNode->count() === 0) {
                throw new \Exception('Form not found on mokuji.htm page.');
            }

            $formAction = $formNode->attr('action');
            $prevValue = $formNode->filter('input[name="prev"]')->attr('value');
            $baseUri = new Uri(self::MANUAL_MOKUJI_URL);
            $absoluteNextPageUrl = (string) UriResolver::resolve($baseUri, new Uri($formAction));

            $this->client->post($absoluteNextPageUrl, ['form_params' => ['prev' => $prevValue]]);

            $this->searchAndDownloadCsv();
            $this->eyachoData_1();
            $this->eyachoData_2();
            log::info('Successfully navigated and prepared for data download.');
        } catch (GuzzleException $e) {
            throw new \Exception("Error accessing manual pages: " . $e->getMessage(), 0, $e);
        } catch (\Exception $e) {
            throw $e; // Re-throw specific exceptions caught within this method
        }
    }

    private function searchAndDownloadCsv(): void
    {
        try {
            $response = $this->client->post(self::ADDR103_URL, ['form_params' => $this->getSearchFormParamsPhoneBook()]);

            $this->handleCsvDownload($response, $this->boxFolderIdPhoneBookData, '電話帳Data_');
            log::info('Phone book data CSV downloaded and uploaded.');
        } catch (GuzzleException $e) {
            throw new \Exception("Error during phone book CSV search and download: " . $e->getMessage(), 0, $e);
        }
    }

    private function eyachoData_1(): void
    {
        try {
            $response = $this->client->post(self::ADDR103_URL, ['form_params' => $this->getSearchFormParamsOrgInfo()]);
            $this->handleCsvDownload($response, $this->boxFolderIdOrgInfo, '組織情報_');
            log::info('Organization info data CSV (type 1) downloaded and uploaded.');
        } catch (GuzzleException $e) {
            throw new \Exception("Error during organization info CSV (type 1) search and download: " . $e->getMessage(), 0, $e);
        }
    }

    private function eyachoData_2(): void
    {
        try {
            $response = $this->client->post(self::ADDR103_URL, ['form_params' => $this->getSearchFormParamsOrgInfoDoboku()]);
            $this->handleCsvDownload($response, $this->boxFolderIdOrgInfoDoboku, '組織情報＿土木_');
            log::info('Organization info data CSV (type 2) downloaded and uploaded.');
        } catch (GuzzleException $e) {
            throw new \Exception("Error during organization info CSV (type 2) search and download: " . $e->getMessage(), 0, $e);
        }
    }

    private function handleCsvDownload($response, string $folderId, string $filePrefix): void
    {
        if ($response->getStatusCode() === 200) {
            $contentType = $response->getHeaderLine('Content-Type');
            if (strpos($contentType, 'text/csv') !== false || strpos($contentType, 'application/octet-stream') !== false) {
                $csvData = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'shift_jis');
                $fileName = $filePrefix . date('Ymd') . '.csv';
                $this->uploadToBox($folderId, $csvData, $fileName);
                dump("{$fileName} downloaded and uploaded to Box successfully.");
            } else {
                throw new \Exception("Expected CSV data in response, but got: " . $contentType);
            }
        } else {
            throw new \Exception("Error submitting search on ADDR102.aspx: " . $response->getStatusCode());
        }
    }

    /**
     * Uploads a file to Box.
     *
     * @param string $upload_file
     * @param string $upload_file_name
     * @param string $folder_id
     * @return void
     * @throws \Exception
     */
    private function uploadToBox(string $folder_id, string $upload_file, string $upload_file_name): void
    {
        try {
            $request_url = 'https://upload.box.com/api/2.0/files/content';
            // $access_token = session('access_token'); // Ensure this session variable is set
            if (!$this->box_access_token) {
                throw new \Exception('Box access token not found in session.');
            }

            $header = [
                "Authorization" => "Bearer " . $this->box_access_token,
            ];
            $multipart =  [
                [
                    'name'     => 'attributes',
                    'contents' => json_encode([
                        'name'      => $upload_file_name,
                        'parent'    => ['id' => $folder_id]
                    ])
                ],
                [
                    'name' => 'file',
                    'contents' => $upload_file,
                    'filename' => $upload_file_name,
                ]
            ];

            $client = new \GuzzleHttp\Client();
            $response = $client->request('POST', $request_url, [
                'headers'   => $header,
                'multipart' => $multipart
            ]);

            if ($response->getStatusCode() < 200 || $response->getStatusCode() >= 300) {
                throw new \Exception('Failed to upload file to Box. Status Code: ' . $response->getStatusCode() . ' Body: ' . $response->getBody());
            }
            log::info("File '{$upload_file_name}' uploaded to Box folder '{$folder_id}' successfully.");
        } catch (GuzzleException $e) {
            throw new \Exception("Error uploading to Box: " . $e->getMessage(), 0, $e);
        } catch (\Exception $e) {
            throw $e; // Re-throw other exceptions
        }
    }
