<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Common\Box;
use Illuminate\Http\Request;

use GuzzleHttp\Cookie\CookieJar;
use GuzzleHttp\Client;
use GuzzleHttp\Exception\GuzzleException;
use Symfony\Component\DomCrawler\Crawler;
use GuzzleHttp\Psr7\Uri;
use GuzzleHttp\Psr7\UriResolver;
use App\Models\LoginModel;

class EmailDataScraperController extends Controller
{
    private $cookieJar;
    private $client;
    private $username = '53439'; // 実際のユーザー名に置き換えてください
    private $password = 'daiD627'; // 実際のパスワードに置き換えてください
    private $groupCompanyCode = 'U'; // 必要なグループ会社コードに置き換えてください
    // private $boxAccessToken = 'YOUR_BOX_ACCESS_TOKEN'; // あなたのBoxのアクセストークンに置き換えてください
    // private $boxFolderId = '322230918978'; // 保存先のBoxフォルダIDに置き換えてください
    // private $boxFolderId = '322352598808'; // 保存先のBoxフォルダIDに置き換えてください
    private $boxFolderIdPhoneBookData = '322352598808';
    private $boxFolderIdOrgInfo = '322352598808';

    private $boxFolderIdOrgInfo_doboku = '322352598808';

    public function __construct()
    {
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
                // 'Referer' => 'http://it-nw.isc.obayashi.co.jp/pick/manual/mokuji.htm',
                'Referer' => 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR101.aspx',
                'Upgrade-Insecure-Requests' => '1',
            ],
        ]);
    }
    public function scrapeAndStoreEmailData()
    {

        try {
            $this->login();
        } catch (\Exception $e) {
            return 'Error: ' . $e->getMessage();
        }
        return 'CSV file downloaded and uploaded to Box successfully.';


        // return view('協力会社管理.partnerCompanyInfo');
    }

    public function login()
    {
        $loginUrl = 'http://login.fc.obayashi.co.jp/sso/UI/Login';
        $successRedirectUrl = 'http://www.fc.obayashi.co.jp/';

        try {
            $response = $this->client->get($loginUrl);
            $loginData = [
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

            $response = $this->client->post($loginUrl, [
                'form_params' => $loginData,
            ]);

            if ($response->getStatusCode() >= 200 && $response->getStatusCode() < 300) {
                $this->callFinalUrl();
            } else {
                $loginHtml = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'SJIS-win');
                throw new \Exception('Login failed: ' . $loginHtml);
            }
        } catch (GuzzleException $e) {
            throw new \Exception('An error occurred during login: ' . $e->getMessage());
        }
    }

    private function callFinalUrl()
    {
        try {
            $this->client->get("http://it-nw.isc.obayashi.co.jp/pick/manual/addr_man.htm");
            $mokujiResponse = $this->client->get("http://it-nw.isc.obayashi.co.jp/pick/manual/mokuji.htm");
            $mokujiHtml = mb_convert_encoding((string) $mokujiResponse->getBody(), 'UTF-8', 'shift_jis');
            $mokujiCrawler = new Crawler($mokujiHtml);
            $formNode = $mokujiCrawler->filter('form[name="f"]');
            $formAction = $formNode->attr('action');
            $prevValue = $formNode->filter('input[name="prev"]')->attr('value');
            $baseUri = new Uri("http://it-nw.isc.obayashi.co.jp/pick/manual/");
            $absoluteNextPageUrl = (string) UriResolver::resolve($baseUri, new Uri($formAction));
            $this->client->post($absoluteNextPageUrl, ['form_params' => ['prev' => $prevValue]]);
            $this->searchAndDownloadCsv();
            $this->eyachoData_1();
            $this->eyachoData_2();
        } catch (GuzzleException $e) {
            throw new \Exception("Error accessing manual pages: " . $e->getMessage());
        }
    }

    private function searchAndDownloadCsv()
    {
        $searchUrl1 = 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR001.aspx';
        $this->client->post($searchUrl1, [
            'form_params' => [
                'chkJG_002' => '1', // 現場
                'chkKKubun_001' => '1', // 国内土木
                'chkKKubun_003' => '1', // 国内建築
            ],
        ]);

        $searchUrl2 = 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR002.aspx';
        $this->client->post($searchUrl2, [
            'form_params' => [
                'chkKojinCode' => '1', // 個人コード
                'chkKojinName' => '0', // 氏名 (チェックしない)
            ],
        ]);

        $searchUrl3 = 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR003.aspx';
        $response = $this->client->post($searchUrl3);
        if ($response->getStatusCode() !== 200) {
            throw new \Exception("Error submitting search on ADDR003.aspx: " . $response->getStatusCode());
        }

        $searchResultUrl = 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR103.aspx';
        $response = $this->client->post($searchResultUrl, [
            'form_params' => [
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
                'chkJG_002' => '1',
                'chkKKubun_001' => '1',
                'chkKKubun_002' => '0',
                'chkKKubun_003' => '1',
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
            ],
        ]);

        if ($response->getStatusCode() === 200) {
            $contentType = $response->getHeaderLine('Content-Type');
            if (strpos($contentType, 'text/csv') !== false || strpos($contentType, 'application/octet-stream') !== false) {
                $csvData = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'shift_jis');
                $fileName = '電話帳Data_' . date('Ymd') . '.csv';
                $this->uploadToBox($this->boxFolderIdPhoneBookData, $csvData, $fileName);
                // $box = new Box();
                // $box->upload_file($this->boxFolderId, $csvData, 'address_search_result.csv');
                dump('CSV file downloaded and uploaded to Box successfully.');
            } else {
                throw new \Exception("Expected CSV data in ADDR102.aspx response, but got: " . $contentType);
            }
        } else {
            throw new \Exception("Error submitting search on ADDR102.aspx: " . $response->getStatusCode());
        }
    }

    private function eyachoData_1()
    {
        $searchResultUrl = 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR103.aspx';
        $response = $this->client->post($searchResultUrl, [
            'form_params' => [
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
            ],
        ]);

        if ($response->getStatusCode() === 200) {
            $contentType = $response->getHeaderLine('Content-Type');
            if (strpos($contentType, 'text/csv') !== false || strpos($contentType, 'application/octet-stream') !== false) {
                $csvData = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'shift_jis');
                $fileName = '組織情報_' . date('Ymd') . '.csv';
                $this->uploadToBox($this->boxFolderIdOrgInfo, $csvData, $fileName);
                dump('CSV file downloaded and uploaded to Box successfully.');
            } else {
                throw new \Exception("Expected CSV data in ADDR102.aspx response, but got: " . $contentType);
            }
        } else {
            throw new \Exception("Error submitting search on ADDR102.aspx: " . $response->getStatusCode());
        }
    }

    private function eyachoData_2()
    {
        $searchResultUrl = 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR103.aspx';
        $response = $this->client->post($searchResultUrl, [
            'form_params' => [
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
                'chkHonKen_001' => '1',
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
                'chkSyoku_002' => '1',
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
            ],
        ]);

        if ($response->getStatusCode() === 200) {
            $contentType = $response->getHeaderLine('Content-Type');
            if (strpos($contentType, 'text/csv') !== false || strpos($contentType, 'application/octet-stream') !== false) {
                $csvData = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'shift_jis');
                $fileName = '組織情報＿土木_' . date('Ymd') . '.csv';
                $this->uploadToBox($this->boxFolderIdOrgInfo_doboku, $csvData, $fileName);
                // $box = new Box();
                // $box->upload_file($this->boxFolderId, $csvData, 'address_search_result.csv');
                dump('CSV file downloaded and uploaded to Box successfully.');
            } else {
                throw new \Exception("Expected CSV data in ADDR102.aspx response, but got: " . $contentType);
            }
        } else {
            throw new \Exception("Error submitting search on ADDR102.aspx: " . $response->getStatusCode());
        }
    }

    private  function uploadToBox($folder_id, $upload_file, $upload_file_name)
    {
        try {
            $request_url    = 'https://upload.box.com/api/2.0/files/content';
            $access_token   = session('access_token');
            $header         = [
                "Authorization" => "Bearer " . $access_token,
            ];
            $multipart      =  [
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
            $client         = new \GuzzleHttp\Client();
            $response       = $client->request('POST', $request_url, [
                'headers'   => $header,
                'multipart' => $multipart
            ]);
        } catch (GuzzleException $e) {
            throw new \Exception($e);
        }
    }
}
