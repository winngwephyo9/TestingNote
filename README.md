![image](https://github.com/user-attachments/assets/6e2195de-4fa7-4d79-9144-8d283cb7a7f4)



![image](https://github.com/user-attachments/assets/3752af46-9662-4f31-9872-568213bb4aa0)
![image](https://github.com/user-attachments/assets/d75e8614-7059-4d94-94ae-19d3e4532222)



![image](https://github.com/user-attachments/assets/d4838317-d8a3-4ba7-86ca-6eb1e6f041a4)



ログの名前:         Microsoft-Windows-TaskScheduler/Operational
ソース:           Microsoft-Windows-TaskScheduler
日付:            2025/06/02 14:20:21
イベント ID:       201
タスクのカテゴリ:      操作が完了しました
レベル:           情報
キーワード:         
ユーザー:          SYSTEM
コンピューター:       K22080031.o365-obayashi.co.jp
説明:
タスク スケジューラは、タスク "\EmailDataScraper\ScrapeEmailData"、インスタンス "{3b4b3bd4-b907-4763-b91b-54b353c97e17}"、操作 "C:\xampp\php\php.exe" を正常に完了しました。リターン コード: 2147942401
イベント XML:
<Event xmlns="http://schemas.microsoft.com/win/2004/08/events/event">
  <System>
    <Provider Name="Microsoft-Windows-TaskScheduler" Guid="{de7b24ea-73c8-4a09-985d-5bdadcfa9017}" />
    <EventID>201</EventID>
    <Version>2</Version>
    <Level>4</Level>
    <Task>201</Task>
    <Opcode>2</Opcode>
    <Keywords>0x8000000000000000</Keywords>
    <TimeCreated SystemTime="2025-06-02T05:20:21.4031479Z" />
    <EventRecordID>67665</EventRecordID>
    <Correlation ActivityID="{3b4b3bd4-b907-4763-b91b-54b353c97e17}" />
    <Execution ProcessID="2548" ThreadID="17812" />
    <Channel>Microsoft-Windows-TaskScheduler/Operational</Channel>
    <Computer>K22080031.o365-obayashi.co.jp</Computer>
    <Security UserID="S-1-5-18" />
  </System>
  <EventData Name="ActionSuccess">
    <Data Name="TaskName">\EmailDataScraper\ScrapeEmailData</Data>
    <Data Name="TaskInstanceId">{3b4b3bd4-b907-4763-b91b-54b353c97e17}</Data>
    <Data Name="ActionName">C:\xampp\php\php.exe</Data>
    <Data Name="ResultCode">2147942401</Data>
    <Data Name="EnginePID">14200</Data>
  </EventData>
</Event>
![Uploading image.png…]()


C:\xampp\php\php.exe


![image](https://github.com/user-attachments/assets/51d10f1b-3f99-4f90-b1bc-340324862498)
<?php

namespace App\Console\Commands;

use App\Http\Controllers\EmailDataScraperController;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\Log;

class ScrapeEmailDataCommand extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'scrape:email-data';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Scrapes email data and stores it locally and in Box.';

    /**
     * Execute the console command.
     */
    public function handle()
    {
        Log::info('ScrapeEmailDataCommand started.');
        try {
            $controller = new EmailDataScraperController();
            $controller->scrapeAndStoreEmailData();
            $this->info('Email data scraped and stored successfully.');
            Log::info('ScrapeEmailDataCommand finished successfully.');
        } catch (\Exception $e) {
            $this->error('Error scraping email data: ' . $e->getMessage());
            Log::error('ScrapeEmailDataCommand failed: ' . $e->getMessage(), ['exception' => $e]);
        }
    }
}



<?php

namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;

class Kernel extends ConsoleKernel
{
    /**
     * Define the application's command schedule.
     */
    protected function schedule(Schedule $schedule): void
    {
        // Run the email data scraper every Monday at 3:00 AM (adjust time as needed)
        $schedule->command('scrape:email-data')->weeklyOn(1, '03:00'); // 1 = Monday
    }

    /**
     * Register the commands for the application.
     */
    protected function commands(): void
    {
        $this->load(__DIR__.'/Commands');

        require base_path('routes/console.php');
    }
}









Route::get('/scrape-email-data', [EmailDataScraperController::class, 'scrapeAndStoreEmailData']);

class EmailDataScraperController extends Controller
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
            // $this->box_access_token = $this->getBoxAccessToken(); // Get token dynamically
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

                $filePath = 'C:\Users\UHR757\ScrappingDataTesting\\' . $fileName; // Make sure this folder exists and is writable
                file_put_contents($filePath, $csvData);

                // $this->uploadToBox($folderId, $csvData, $fileName);
                dump("{$fileName} downloaded and uploaded to Box successfully.");
            } else {
                throw new \Exception("Expected CSV data in response, but got: " . $contentType);
            }
        } else {
            throw new \Exception("Error submitting search on ADDR102.aspx: " . $response->getStatusCode());
        }
    }
}
