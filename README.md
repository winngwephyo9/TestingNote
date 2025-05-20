


<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title>メールアドレス・電話番号　検索</title>
<script>
    function onCSVDownLoad() {
        document.form1.submit();
        return false;
    }
</script>
<style type="text/css">
  table { font-size:12px; }
  #jyouken { background-color:#333366;width:100%; }
  #jyouken th { text-align:left;border:none;background-color:#ccccff;font-weight:normal;vertical-align:top; }
  #jyouken th.output { background-color:#666666;color:#ffffff; }
  #jyouken td { border:none;background-color:#dddddd;vertical-align:top; }
  .s1 { color:navy;font-size:13px; }
</style>
</head>
<body>
<div>


以下の条件で検索しますがよろしいですか。<br />

<table id="jyouken" cellpadding="0" cellspacing="1">
  <colgroup>
    <col style="width:40%" />
    <col style="width:60%" />
  </colgroup>
  <tr><th>支店</th><td style="word-break:break-all;"></td></tr>
  <tr><th>組織コード</th><td style="word-break:break-all;"></td></tr>
  <tr><th>組織名</th><td style="word-break:break-all;"></td></tr>
  <tr><th>常設・現場</th><td style="word-break:break-all;">現場</td></tr>
  <tr><th>工事区分</th><td style="word-break:break-all;">国内土木</td></tr>
  <tr><th>個人コード</th><td style="word-break:break-all;"></td></tr>
  <tr>
    <th>協力スタッフ</th>
    <td style="word-break:break-all;">

    </td>
  </tr>
  <tr><th>本務・兼務・他在籍・仮想在籍</th><td style="word-break:break-all;"></td></tr>
  <tr><th>職位（セキュリティランク）</th><td style="word-break:break-all;"> ～ </td></tr>
  <tr>
    <th>役職</th>
    <td style="word-break:break-all;">

    </td>
  </tr>
  <tr>
    <th>従区分名</th>
    <td style="word-break:break-all;">

    </td>
  </tr>
  <tr><th>職種</th><td style="word-break:break-all;"></td></tr>
  <tr><th class="output">出力項目</th><td style="word-break:break-all;">個人コード</td></tr>
</table>
</div>

3913件　検索時間：0.015秒　<br />

<a href="javascript:void(0);" onclick="onCSVDownLoad();">CSV出力</a>

 （クリックするとダウンロードを開始します。）<br />
以上

<form method="POST" id="form1" name="form1" action="ADDR103.aspx" target="_blank">
<input type="hidden" id="chkTen_09" name="chkTen_09" value="0" />
<input type="hidden" id="chkTen_11" name="chkTen_11" value="0" />
<input type="hidden" id="chkTen_15" name="chkTen_15" value="0" />
<input type="hidden" id="chkTen_10" name="chkTen_10" value="0" />
<input type="hidden" id="chkTen_26" name="chkTen_26" value="0" />
<input type="hidden" id="chkTen_12" name="chkTen_12" value="0" />
<input type="hidden" id="chkTen_13" name="chkTen_13" value="0" />
<input type="hidden" id="chkTen_14" name="chkTen_14" value="0" />
<input type="hidden" id="chkTen_16" name="chkTen_16" value="0" />
<input type="hidden" id="chkTen_17" name="chkTen_17" value="0" />
<input type="hidden" id="chkTen_19" name="chkTen_19" value="0" />
<input type="hidden" id="chkTen_27" name="chkTen_27" value="0" />
<!--<input type="hidden" id="chkTen_30" name="chkTen_30" value="0" />-->
<input type="hidden" id="chkTen_31" name="chkTen_31" value="0" />
<input type="hidden" id="chkTen_32" name="chkTen_32" value="0">
<input type="hidden" id="chkTen_46" name="chkTen_46" value="0" />
<input type="hidden" id="chkTen_36" name="chkTen_36" value="0" />
<input type="hidden" id="chkTen_66" name="chkTen_66" value="0" />
<input type="hidden" id="txtSosikiCode" name="txtSosikiCode" value="" />
<input type="hidden" id="txtSosikiName" name="txtSosikiName" value="" />
<input type="hidden" id="chkJG_001" name="chkJG_001" value="0" />
<input type="hidden" id="chkJG_002" name="chkJG_002" value="1" />
<input type="hidden" id="chkKKubun_001" name="chkKKubun_001" value="1" />
<input type="hidden" id="chkKKubun_002" name="chkKKubun_002" value="0" />
<input type="hidden" id="chkKKubun_003" name="chkKKubun_003" value="0" />
<input type="hidden" id="chkKKubun_004" name="chkKKubun_004" value="0" />
<input type="hidden" id="txtKojinCode" name="txtKojinCode" value="" />
<input type="hidden" id="rdoKyoryoku" name="rdoKyoryoku" value="0" />
<input type="hidden" id="txtKyoryoku" name="txtKyoryoku" value="" />
<input type="hidden" id="chkHonKen_001" name="chkHonKen_001" value="0" />
<input type="hidden" id="chkHonKen_002" name="chkHonKen_002" value="0" />
<input type="hidden" id="chkHonKen_003" name="chkHonKen_003" value="0" />
<input type="hidden" id="chkHonKen_004" name="chkHonKen_004" value="0" />
<input type="hidden" id="ddtSecRankS" name="ddtSecRankS" value="" />
<input type="hidden" id="ddtSecRankE" name="ddtSecRankE" value="" />
<input type="hidden" id="rdoYaku" name="rdoYaku" value="0" />
<input type="hidden" id="txtYaku" name="txtYaku" value="" />
<input type="hidden" id="rdoJuKubun" name="rdoJuKubun" value="0" />
<input type="hidden" id="txtJuKubun" name="txtJuKubun" value="" />
<input type="hidden" id="chkSyoku_001" name="chkSyoku_001" value="0" />
<input type="hidden" id="chkSyoku_002" name="chkSyoku_002" value="0" />
<input type="hidden" id="chkSyoku_003" name="chkSyoku_003" value="0" />
<input type="hidden" id="chkSyoku_004" name="chkSyoku_004" value="0" />
<input type="hidden" id="chkSyoku_005" name="chkSyoku_005" value="0" />
<input type="hidden" id="chkSyoku_006" name="chkSyoku_006" value="0" />

<input type="hidden" id="hidKojinCode" name="hidKojinCode" value="1" />
<input type="hidden" id="hidHonken" name="hidHonken" value="0" />
<input type="hidden" id="hidKojinName" name="hidKojinName" value="0" />
<input type="hidden" id="hidKojinKana" name="hidKojinKana" value="0" />
<input type="hidden" id="hidRank" name="hidRank" value="0" />
<input type="hidden" id="hidMailAddress" name="hidMailAddress" value="0" />
<input type="hidden" id="hidYaku" name="hidYaku" value="0" />
<input type="hidden" id="hidTenCode" name="hidTenCode" value="0" />
<input type="hidden" id="hidJukubun" name="hidJukubun" value="0" />
<input type="hidden" id="hidTenMei" name="hidTenMei" value="0" />
<input type="hidden" id="hidJukubunName" name="hidJukubunName" value="0" />
<input type="hidden" id="hidSosikiCode" name="hidSosikiCode" value="0" />
<input type="hidden" id="hidSyoku" name="hidSyoku" value="0" />
<input type="hidden" id="hidSosikiMei" name="hidSosikiMei" value="0" />
<input type="hidden" id="hidNaisen" name="hidNaisen" value="0" />
<input type="hidden" id="hidJGKubun" name="hidJGKubun" value="0" />
<input type="hidden" id="hidGaisen" name="hidGaisen" value="0" />
<input type="hidden" id="hidKoujiKubun" name="hidKoujiKubun" value="0" />
<input type="hidden" id="hidWorksite" name="hidWorksite" value="0" />
</form>


</body>
</html>


Expected CSV data in ADDR102.aspx response, but got: text/html; charset=utf-8

if ($response->getStatusCode() === 200) {
            $contentType = $response->getHeaderLine('Content-Type');
            if (strpos($contentType, 'text/csv') !== false || strpos($contentType, 'application/octet-stream') !== false) {
                $csvData = (string) $response->getBody();
                $this->uploadToBox($csvData, 'address_search_result.csv');
                return 'CSV file downloaded and uploaded to Box successfully.';
            } else {
                throw new \Exception("Expected CSV data in ADDR102.aspx response, but got: " . $contentType);
            }
        } else {
            throw new \Exception("Error submitting search on ADDR102.aspx: " . $response->getStatusCode());
        }
        
        
        
        <?php

namespace App\Http\Controllers;

use GuzzleHttp\Client;
use GuzzleHttp\Cookie\CookieJar;
use GuzzleHttp\Exception\GuzzleException;
use Symfony\Component\DomCrawler\Crawler;
use GuzzleHttp\Psr7\Uri;
use GuzzleHttp\UriTemplate\UriTemplate;
use Illuminate\Support\Facades\Storage;

class ObayashiAddressSearchController extends Controller
{
    private $cookieJar;
    private $client;
    private $username = '53439'; // 実際のユーザー名に置き換えてください
    private $password = 'daiD627'; // 実際のパスワードに置き換えてください
    private $groupCompanyCode = 'U'; // 必要なグループ会社コードに置き換えてください
    private $boxAccessToken = 'YOUR_BOX_ACCESS_TOKEN'; // あなたのBoxのアクセストークンに置き換えてください
    private $boxFolderId = 'YOUR_BOX_FOLDER_ID'; // 保存先のBoxフォルダIDに置き換えてください

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
                'Referer' => 'http://it-nw.isc.obayashi.co.jp/pick/manual/mokuji.htm',
                'Upgrade-Insecure-Requests' => '1',
            ],
        ]);
    }

    public function run()
    {
        try {
            $this->login();
        } catch (\Exception $e) {
            return 'Error: ' . $e->getMessage();
        }
        return 'CSV file downloaded and uploaded to Box successfully.';
    }

    private function login()
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

        $searchResultUrl = 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR102.aspx';
        $response = $this->client->post($searchResultUrl, [
            'form_params' => [
                'chkTen_09' => '0', 'chkTen_11' => '0', 'chkTen_15' => '0', 'chkTen_10' => '0', 'chkTen_26' => '0', 'chkTen_12' => '0', 'chkTen_13' => '0', 'chkTen_14' => '0', 'chkTen_16' => '0', 'chkTen_17' => '0', 'chkTen_19' => '0', 'chkTen_27' => '0',
                'chkTen_31' => '0', 'chkTen_32' => '0', 'chkTen_46' => '0', 'chkTen_36' => '0', 'chkTen_66' => '0',
                'txtSosikiCode' => '', 'txtSosikiName' => '',
                'chkJG_001' => '0', 'chkJG_002' => '1',
                'chkKKubun_001' => '1', 'chkKKubun_002' => '0', 'chkKKubun_003' => '1', 'chkKKubun_004' => '0',
                'txtKojinCode' => '', 'rdoKyoryoku' => '0', 'txtKyoryoku' => '',
                'chkHonKen_001' => '0', 'chkHonKen_002' => '0', 'chkHonKen_003' => '0', 'chkHonKen_004' => '0',
                'ddtSecRankS' => '', 'ddtSecRankE' => '', 'rdoYaku' => '0', 'txtYaku' => '', 'rdoJuKubun' => '0', 'txtJuKubun' => '',
                'chkSyoku_001' => '0', 'chkSyoku_002' => '0', 'chkSyoku_003' => '0', 'chkSyoku_004' => '0', 'chkSyoku_005' => '0', 'chkSyoku_006' => '0',
                'hidKojinCode' => '1', 'hidHonken' => '0', 'hidKojinName' => '0', 'hidKojinKana' => '0', 'hidRank' => '0', 'hidMailAddress' => '0', 'hidYaku' => '0', 'hidTenCode' => '0', 'hidJukubun' => '0', 'hidTenMei' => '0', 'hidJukubunName' => '0', 'hidSosikiCode' => '0', 'hidSyoku' => '0', 'hidSosikiMei' => '0', 'hidNaisen' => '0', 'hidJGKubun' => '0', 'hidGaisen' => '0', 'hidKoujiKubun' => '0', 'hidWorksite' => '0',
            ],
        ]);

        if ($response->getStatusCode() !== 200) {
            throw new \Exception("Error submitting search on ADDR102.aspx: " . $response->getStatusCode());
        }
        $body = (string) $response->getBody();
        if (preg_match('/<a href="javascript:void\(0\);" onclick="onCSVDownLoad\(\);">CSV出力<\/a>/', $body)) {
            $csvDownloadUrl = 'http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR103.aspx';
            $csvResponse = $this->client->post($csvDownloadUrl, [
                'form_params' => [
                    'chkTen_09' => '0', 'chkTen_11' => '0', 'chkTen_15' => '0', 'chkTen_10' => '0', 'chkTen_26' => '0', 'chkTen_12' => '0', 'chkTen_13' => '0', 'chkTen_14' => '0', 'chkTen_16' => '0', 'chkTen_17' => '0', 'chkTen_19' => '0', 'chkTen_27' => '0',
                    'chkTen_31' => '0', 'chkTen_32' => '0', 'chkTen_46' => '0', 'chkTen_36' => '0', 'chkTen_66' => '0',
                    'txtSosikiCode' => '', 'txtSosikiName' => '',
                    'chkJG_001' => '0', 'chkJG_002' => '1',
                    'chkKKubun_001' => '1', 'chkKKubun_002' => '0', 'chkKKubun_003' => '1', 'chkKKubun_004' => '0',
                    'txtKojinCode' => '', 'rdoKyoryoku' => '0', 'txtKyoryoku' => '',
                    'chkHonKen_001' => '0', 'chkHonKen_002' => '0', 'chkHonKen_003' => '0', 'chkHonKen_004' => '0',
                    'ddtSecRankS' => '', 'ddtSecRankE' => '', 'rdoYaku' => '0', 'txtYaku' => '', 'rdoJuKubun' => '0', 'txtJuKubun' => '',
                    'chkSyoku_001' => '0', 'chkSyoku_002' => '0', 'chkSyoku_003' => '0', 'chkSyoku_004' => '0', 'chkSyoku_005' => '0', 'chkSyoku_006' => '0',
                    'hidKojinCode' => '1', 'hidHonken' => '0', 'hidKojinName' => '0', 'hidKojinKana' => '0', 'hidRank' => '0', 'hidMailAddress' => '0', 'hidYaku' => '0', 'hidTenCode' => '0', 'hidJukubun' => '0', 'hidTenMei' => '0', 'hidJukubunName' => '0', 'hidSosikiCode' => '0', 'hidSyoku' => '0', 'hidSosikiMei' => '0', 'hidNaisen' => '0', 'hidJGKubun' => '0', 'hidGaisen' => '0', 'hidKoujiKubun' => '0', 'hidWorksite' => '0',
                ],
            ]);

            if ($csvResponse->getStatusCode() === 200) {
                $csvData = $csvResponse->getBody();
                $this->uploadToBox($csvData, 'address_search_result.csv');
                return 'CSV file downloaded and uploaded to Box successfully.';
            } else {
                throw new \Exception("Error downloading CSV file: " . $csvResponse->getStatusCode());
            }
        } else {
            throw new \Exception("CSV download link not found.");
        }
    }

    private function uploadToBox(string $fileContent, string $fileName)
    {
        $boxApiUrl = 'https://upload.box.com/api/2.0/files/content';
        $headers = [
            'Authorization' => 'Bearer ' . $this->boxAccessToken,
        ];
        $multipart = [
            [
                'name' => 'attributes',
                'contents' => json_encode([
                    'name' => $fileName,
                    'parent' => ['id' => $this->boxFolderId],
                ]),
            ],
            [
                'name' => 'file',
                'contents' => $fileContent,
                'filename' => $fileName,
            ],
        ];

        try {
            $response = $this->client->request('POST', $boxApiUrl, [
                'headers' => $headers,
                'multipart' => $multipart,
            ]);

            if ($response->getStatusCode() !== 201) {
                throw new \Exception('Failed to upload file to Box: ' . $response->getStatusCode() . ' - ' . $response->getBody());
            }
        } catch (GuzzleException $e) {
            throw new \Exception('Error communicating with Box API: ' . $e->getMessage());
        }
    }
}


source code
   private $cookieJar;
    private $client;
    public function __construct()
    {
        $this->cookieJar = new CookieJar();
        $this->client = new Client([
            'cookies' => $this->cookieJar, // Use the explicit CookieJar
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
                'Referer' => 'http://it-nw.isc.obayashi.co.jp/pick/manual/mokuji.htm',
                // 'Referer' => 'http://login.fc.obayashi.co.jp/sso/UI/Login?type=obayashi&goto=http%3A%2F%2Fintralogin.fc.obayashi.co.jp%2Fcgi-bin%2Fclogin.cgi%3Faccess%3Dhttp%253A%252F%252Fwww.fc.obayashi.co.jp%252F%252F',
                'Upgrade-Insecure-Requests' => '1',
            ],
        ]);
    }
    public function login()
    {
        $loginUrl = 'http://login.fc.obayashi.co.jp/sso/UI/Login';
        $successRedirectUrl = 'http://www.fc.obayashi.co.jp/';
        $username = '53439'; // Replace with the actual username
        $password = 'daiD627'; // Replace with the actual password
        $groupCompanyCode = 'U'; // Replace with the desired group company code (e.g., 'U' for 大林組)

        try {
            // 1. Get the initial login page
            $response = $this->client->get($loginUrl);
            $html = (string) $response->getBody();

            // 2. Prepare the login form data
            $loginData = [
                'IDToken0-0' => $groupCompanyCode,
                'IDToken0' => $groupCompanyCode,
                'IDToken1' => $groupCompanyCode . $username,
                'IDToken2' => $password,
                'Login.Submit' => 'ログイン',
                'goto' => 'aHR0cDovL2ludHJhbG9naW4uZmMub2JheWFzaGkuY28uanAvY2dpLWJpbi9jbG9naW4uY2dpP2FjY2Vzcz1odHRwOi8vd3d3LmZjLm9iYXlhc2hpLmNvLmpwLw==',
                'gotoOnFail' => '',
                'SunQueryParamsString' => 'dHlwZT1vYmF5YXNoaQ==',
                'encoded' => 'true',
                'type' => 'obayashi',
                'gx_charset' => 'UTF-8',
                'IDButton' => 'ログイン',
            ];

            // 3. Submit the login form
            $response = $this->client->post($loginUrl, [
                'form_params' => $loginData,
            ]);
            // dump("Login Status Code:", $response->getStatusCode());

            if ($response->getStatusCode() >= 200 && $response->getStatusCode() < 300) {
                // Optionally, make a subsequent request to the redirected URL to ensure login success
                $successPageResponse = $this->client->get($successRedirectUrl);
                // You might want to check the content of $successPageResponse here to confirm login
                // dd(vars: $this->cookieJar->toArray());
                $this->callFinalUrl();
            } else {
                // Login failed
                $loginHtml = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'SJIS-win');
                dd('Login failed.', $loginHtml);
            }
        } catch (GuzzleException $e) {
            dd('An error occurred during login: ' . $e->getMessage());
        }
    }

    private function callFinalUrl()
    {
        // **Step 1: Access the initial manual page (sets up the frameset)**
        try {
            $manualResponse = $this->client->get("http://it-nw.isc.obayashi.co.jp/pick/manual/addr_man.htm");
            $manualHtml = (string) $manualResponse->getBody();
            $manualHtml = mb_convert_encoding($manualHtml, 'UTF-8', 'shift_jis');
            $manualPageCrawler = new Crawler($manualHtml);

            // **Step 2: Access the mokuji.htm frame content**
            $mokujiFrameSrc = $manualPageCrawler->filter('frame[name="mokuji"]')->attr('src');
            $mokujiUrl = "http://it-nw.isc.obayashi.co.jp/pick/manual/" . $mokujiFrameSrc;
            $mokujiResponse = $this->client->get($mokujiUrl);
            $mokujiHtml = (string) $mokujiResponse->getBody();
            $mokujiHtml = mb_convert_encoding($mokujiHtml, 'UTF-8', 'shift_jis');
            $mokujiCrawler = new Crawler($mokujiHtml);

            // **Step 3: Access the chuui.htm frame content (the 'naiyou' frame)**
            $naiyouFrameSrc = $manualPageCrawler->filter('frame[name="naiyou"]')->attr('src');
            $naiyouUrl = "http://it-nw.isc.obayashi.co.jp/pick/manual/" . $naiyouFrameSrc;
            $naiyouResponse = $this->client->get($naiyouUrl);
            $naiyouHtml = (string) $naiyouResponse->getBody();
            // We might not need to parse this, just accessing it might be enough

            // **Step 4: Find the form in mokuji.htm and submit it**
            $formNode = $mokujiCrawler->filter('form[name="f"]');

            if ($formNode->count() > 0) {
                $formAction = $formNode->attr('action');
                $prevValue = $formNode->filter('input[name="prev"]')->attr('value');

                $relativeUri = new Uri($formAction);
                $baseUri = new Uri("http://it-nw.isc.obayashi.co.jp/pick/manual/");
                $absoluteNextPageUrl = (string) UriResolver::resolve($baseUri, $relativeUri);

                $formData = ['prev' => $prevValue];

                // Make a POST request to ADDR000.aspx
                $nextPageResponse = $this->client->request('POST', $absoluteNextPageUrl, [
                    'form_params' => $formData,
                ]);

                if ($nextPageResponse->getStatusCode() >= 200 && $nextPageResponse->getStatusCode() < 300) {
                    // Optionally, make a subsequent request to the redirected URL to ensure login success
                    $successPageResponse = $this->client->get("http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR000.aspx");
                    $nextPageHtml = (string) $successPageResponse->getBody();
                    $nextPageHtml = mb_convert_encoding($nextPageHtml, 'UTF-8', 'shift_jis');
                    dump("Response after POST to ADDR000.aspx:", $nextPageHtml);
                    dd("Success  Final Page Status Code:", $successPageResponse);
                } else {
                    // Login failed
                    $nextPageHtml = mb_convert_encoding((string) $nextPageResponse->getBody(), 'UTF-8', 'SJIS-win');
                    dd('failed.', $nextPageHtml);
                }
            } else {
                dd('Error: Form with name "f" not found in mokuji.htm.');
            }
        } catch (GuzzleException $e) {
            dd("Error accessing addr_man.htm: " . $e->getMessage());
        }
    }

Finally I got this html code 

<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title>メールアドレス検索</title>
<script type="text/javascript" language="javascript">
//================================================================================================//
function writePOSTData(obj, url) {
    var strFormContents = '<form method="post" id="form1" name="form1" action="' + url + '" target="_self">' + "\n"
    + '<input type="hidden" name="txtConditionFile" value="">' + "\n"
    + '</form>';
    obj.document.write(strFormContents);
}
//================================================================================================//
</script>
</head>

<frameset cols="66%,*">
    <frame id="jyoken" name="jyoken" src="ADDR001.aspx" noresize Scrolling="auto" />
    <frameset rows="33%,*">
        <frame id="koumoku" name="koumoku" src="ADDR002.aspx" noresize Scrolling="auto" />
        <frame id="gamen" name="gamen" src="ADDR003.aspx" noresize Scrolling="auto" />
    </frameset>

</frameset>

</html>

This is ADDR001.aspx source code from developer tool
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
  <title>メールアドレス検索</title>
<script type="text/javascript" language="javascript" src="../scripts/addr_common.js"></script>
<script type="text/javascript" language="javascript" src="../scripts/ADDR001.js"></script>
<style type="text/css">
  table { font-size:12px; }
  td { vertical-align:top; }
  /* td.title { background-color:#ccccff; } */
  td.koumoku { border-right:1px solid #000000;border-bottom:1px solid #000000;background-color:#ccccff; }
  td.koumoku2 { border-right:1px solid #000000;background-color:#ccccff; }
  td.sentaku { border-bottom:1px solid #000000; }
  a { color:#0000ff; }
  .col { color:#0000ff; }
  .col_sub { color:#000000; }
  .s1 { color:navy;font-size:13px; }
</style>
    <script language="javascript" type="text/javascript">
// <![CDATA[

        function chkTen_09_onclick() {

        }

// ]]>
    </script>
</head>
<body>


<form method="POST" id="form1" name="form1" action="ADDR101.aspx" target="gamen">
<div style="background-color:#ffff00;">
<table cellspacing="0" style="height:25px;">
  <tr><td style="font-size:16px;font-weight:bold;">メールアドレス・電話番号　検索</td></tr>
</table>
</div>
<div>
<table cellspacing="0" style="height:20px;">
  <tr><td style="font-size:12px;">絞り込みはAND検索です。</td></tr>
</table>
</div>

<table>
  <tr>
    <td><input type="button" id="btnNew" name="btnNew" value="新しい検索条件" onclick="btnNew_click();" /></td>
    <td style="text-align:right;"><input type="button" id="btnClose" name="btnClose" value="閉じる" onclick="btnClose_click();" /></td>
    <td></td>
  </tr>
  <tr>
    <td colspan="2"><input type="button" id="btnSave" name="btnSave" value="検索条件／出力項目　の保存" onclick="btnSave_click();" /></td>
    <td><input type="button" id="btnLoad" name="btnLoad" value="検索条件／出力項目　の読込" onclick="btnLoad_click();" /></td>
  </tr>
</table>

<div style="height:700px;">
<table style="width:97.8%">
  <tr>
    <td style="width:50%;vertical-align:top;">
      <table cellspacing="0" style="border:1px solid #000000;width:100%;">
        <colgroup>
          <col style="width:50%" />
          <col style="width:50%" />
        </colgroup>
        <tr>
          <th style="border-right:1px solid #000000;border-bottom:1px solid #000000;color:#008080;">条件</th>
          <th style="border-bottom:1px solid #000000;color:#008080;">条件の内容</th>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">支店</span><br />
            <span class="col_sub">・未選択の場合は全店が対象</span>
          </td>
          <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td><input type="checkbox" id="chkTen_09" name="chkTen_09" value="09"  /><label for="chkTen_09">本社</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_11" name="chkTen_11" value="11"  /><label for="chkTen_11">東京本店・関東支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_15" name="chkTen_15" value="15"  /><label for="chkTen_15">横浜支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_10" name="chkTen_10" value="10"  /><label for="chkTen_10">大阪本店・京都支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_26" name="chkTen_26" value="26"  /><label for="chkTen_26">神戸支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_12" name="chkTen_12" value="12"  /><label for="chkTen_12">名古屋支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_13" name="chkTen_13" value="13"  /><label for="chkTen_13">九州支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_14" name="chkTen_14" value="14"  /><label for="chkTen_14">東北支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_16" name="chkTen_16" value="16"  /><label for="chkTen_16">札幌支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_17" name="chkTen_17" value="17"  /><label for="chkTen_17">広島支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_19" name="chkTen_19" value="19"  /><label for="chkTen_19">四国支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_27" name="chkTen_27" value="27"  /><label for="chkTen_27">北陸支店</label></td></tr>
              <!--<tr><td><input type="checkbox" id="chkTen_30" name="chkTen_30" value="30"  /><label for="chkTen_30">海外支店</label></td></tr>-->
              <tr><td><input type="checkbox" id="chkTen_31" name="chkTen_31" value="31"  /><label for="chkTen_31">アジア支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_32" name="chkTen_32" value="32"  /><label for="chkTen_32">北米支店</label><td></tr>
              <tr><td><input type="checkbox" id="chkTen_46" name="chkTen_46" value="46"  /><label for="chkTen_46">東日本ロボティクスセンター</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_36" name="chkTen_36" value="36"  /><label for="chkTen_36">西日本ロボティクスセンター</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_66" name="chkTen_66" value="66"  /><label for="chkTen_66">出向</label></td></tr>
              <tr>
                <td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.Ten);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td>
              </tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">組織コード</span><br />
            <span class="col_sub">・未入力の場合は全組織が対象</span><br />
            <span class="col_sub">・半角数字で5文字または7文字</span><br />
            <span class="col_sub">・前方一致</span><br />
            <span class="col_sub">・複数の組織コードを入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">&nbsp;&nbsp;&nbsp;&nbsp;例：09117,0911222</span><br />
            <span class="col_sub">・1000文字以内</span>
          </td>
          <td class="sentaku">
            <textarea id="txtSosikiCode" name="txtSosikiCode" rows="2" cols="20" style="height:100px;width:95%;overflow:hidden;ime-mode:disabled;"></textarea>
          </td>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">組織名</span><br />
            <span class="col_sub">・未入力の場合は全組織が対象</span><br />
            <span style="color:#ff0000;">・設定すると、検索時間が長くなるため、なるべく設定しない</span><br />
            <span class="col_sub">・全角</span><br />
            <span class="col_sub">・部分一致</span><br />
            <span class="col_sub">・複数の組織名を入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">&nbsp;&nbsp;&nbsp;&nbsp;例：本社総務部,グローバル</span><br />
            <span class="col_sub">・3000文字以内</span>
          </td>
          <td class="sentaku">
            <textarea id="txtSosikiName" name="txtSosikiName" rows="2" cols="20" style="height:120px;width:95%;overflow:hidden;ime-mode:active;"></textarea>
          </td>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">常設・現場</span><br />
            <span class="col_sub">・未選択の場合は全員が対象</span>
          </td>
          <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td>
                <input type="checkbox" id="chkJG_001" name="chkJG_001"  value="1" /><label for="chkJG_001">常設</label>
                <input type="checkbox" id="chkJG_002" name="chkJG_002"  value="1" /><label for="chkJG_002">現場</label>
              </td></tr>
              <tr>
                <td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.JGKubun);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td>
              </tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="koumoku2">
            <span class="col">工事区分</span><br />
            <span class="col_sub">・未選択の場合は全工事区分が対象</span>
          </td>
          <td>
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td>
                <input type="checkbox" id="chkKKubun_001" name="chkKKubun_001" value="1"  /><label for="chkKKubun_001">国内土木</label>
                <input type="checkbox" id="chkKKubun_002" name="chkKKubun_002" value="1"  /><label for="chkKKubun_002">海外土木</label>
              </td></tr>
              <tr><td>
                <input type="checkbox" id="chkKKubun_003" name="chkKKubun_003" value="1"  /><label for="chkKKubun_003">国内建築</label>
                <input type="checkbox" id="chkKKubun_004" name="chkKKubun_004" value="1"  /><label for="chkKKubun_004">海外建築</label>
              </td></tr>
              <tr>
                <td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.KKubun);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td>
              </tr>
            </table>
          </td>
        </tr>
      </table>
    </td>
    <td style="width:10px;"></td>
    <td>
      <table cellspacing="0" style="border:1px solid #000000;width:100%;">
        <colgroup>
          <col style="width:50%" />
          <col style="width:50%" />
        </colgroup>
        <tr>
          <th style="border-right:1px solid #000000;border-bottom:1px solid #000000;color:#008080;">条件</th>
          <th style="border-bottom:1px solid #000000;color:#008080;">条件の内容</th>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">個人コード</span><br />
            <span class="col_sub">・未入力の場合は全員が対象</span><br />
            <span class="col_sub">・前方一致</span><br />
            <span class="col_sub">・複数の個人コードを入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">&nbsp;&nbsp;&nbsp;&nbsp;例：11111,21</span><br />
            <span class="col_sub">・1000文字以内</span>
          </td>
          <td class="sentaku">
            <textarea id="txtKojinCode" name="txtKojinCode" rows="2" cols="20" style="height:110px;width:95%;overflow:hidden;ime-mode:disabled;" maxlength="100"></textarea>
          </td>
        </tr>
        <tr>
          <td rowspan="2" class="koumoku">
            <span class="col">協力スタッフ</span><br />
            <span class="col_sub">・未選択の場合は全員が対象</span><br />
            <span class="col_sub">・個人コード1文字目</span><br />
            <span class="col_sub">・複数の文字を入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">・例：H,S,M,J,G,X,Y,Z</span><br />
            <span class="col_sub">・50文字以内</span><br />
            「個人コード体系については<a href="http://it.isc.obayashi.co.jp/security/mng_staff/userid.htm" target="_blank">こちら</a>」
          </td>
          <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td><span class="s1">以下の個人コードを</span><br /></td></tr>
              <tr><td>
                <input type="radio" id="rdoKyoryoku01" name="rdoKyoryoku" value="1"  /><label for="rdoKyoryoku01">抽出する</label>
                <input type="radio" id="rdoKyoryoku02" name="rdoKyoryoku" value="2"  /><label for="rdoKyoryoku02">除外する</label>
              </td></tr>
              <tr><td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.Kyoryoku);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td></tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="sentaku">
            <textarea id="txtKyoryoku" name="txtKyoryoku" rows="2" cols="20" style="height:50px;width:95%;overflow:hidden;ime-mode:disabled;"></textarea>
          </td>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">本務・兼務・他在籍・仮想在籍</span><br />
            <span class="col_sub">・未選択の場合は全員が対象</span>
          </td>
          <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td>
                <input type="checkbox" id="chkHonKen_001" name="chkHonKen_001" value="1"  /><label for="chkHonKen_001">本務</label>
                <input type="checkbox" id="chkHonKen_002" name="chkHonKen_002" value="1"  /><label for="chkHonKen_002">兼務</label>
                <input type="checkbox" id="chkHonKen_003" name="chkHonKen_003" value="1"  /><label for="chkHonKen_003">他在籍</label>
              </td></tr>
              <tr><td><input type="checkbox" id="chkHonKen_004" name="chkHonKen_004" value="1"  /><label for="chkHonKen_004">仮想在籍</label></td></tr>
              <tr><td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.HonKen);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td></tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">職位（セキュリティランク）</span><br />
            <span class="col_sub">・未選択の場合は全職位が対象</span><br />
            「職位については<a href="http://it.isc.obayashi.co.jp/ct_sys/secure_web/secrank.htm" target="_blank">こちら</a>」<br />
            <span class="col_sub">・本務・兼務・他在籍・仮想在籍の条件が未選択の場合、兼務・他在籍・仮想在籍の職位情報も検索範囲になります。</span>
          </td>
          <td class="sentaku">
            <select id="ddtSecRankS" name="ddtSecRankS" style="width:160px;">
            <option value="" selected ></option>
<option value="1">役員・理事</option>
<option value="3">部長・所長</option>
<option value="4">副部長・課長</option>
<option value="5">副課長</option>
<option value="7">主任・職員</option>
<option value="8">期間職員・期間雇用</option>
<option value="11">ｽﾀｯﾌ(従業員同等)</option>
<option value="13">ｽﾀｯﾌ(業務ｼｽﾃﾑ利用)</option>
<option value="14">ｽﾀｯﾌ(電話帳・ｱﾄﾞﾚｽ帳)</option>
<option value="15">ｽﾀｯﾌ(電子ﾒｰﾙのみ)</option>

            </select>～<br />
            <select id="ddtSecRankE" name="ddtSecRankE" style="width:160px;">
            <option value="" selected ></option>
<option value="1">役員・理事</option>
<option value="3">部長・所長</option>
<option value="4">副部長・課長</option>
<option value="5">副課長</option>
<option value="7">主任・職員</option>
<option value="8">期間職員・期間雇用</option>
<option value="11">ｽﾀｯﾌ(従業員同等)</option>
<option value="13">ｽﾀｯﾌ(業務ｼｽﾃﾑ利用)</option>
<option value="14">ｽﾀｯﾌ(電話帳・ｱﾄﾞﾚｽ帳)</option>
<option value="15">ｽﾀｯﾌ(電子ﾒｰﾙのみ)</option>

            </select>
          </td>
        </tr>
        <tr>
          <td rowspan="2" class="koumoku">
            <span class="col">役職</span><br />
            <span class="col_sub">・未選択の場合は全役職が対象</span><br />
            <span class="col_sub">・部分一致</span><br />
            <span class="col_sub">・複数の役職を入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">&nbsp;&nbsp;&nbsp;&nbsp;例：部長,課長</span><br />
            <span class="col_sub">・300文字以内</span>
            </td>
            <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td><span class="s1">以下の役職を</span></td></tr>
              <tr><td>
                <input type="radio" id="rdoYaku01" name="rdoYaku" value="1"  /><label for="rdoYaku01">抽出する</label>
                <input type="radio" id="rdoYaku02" name="rdoYaku" value="2"  /><label for="rdoYaku02">除外する</label>
              </td></tr>
              <tr><td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.Yaku);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td></tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="sentaku">
            <textarea id="txtYaku" name="txtYaku" rows="2" cols="20" style="height:50px;width:95%;overflow:hidden;ime-mode:active;"></textarea>
          </td>
        </tr>
        <tr>
          <td rowspan="2" class="koumoku">
            <span class="col">従区分名</span><br />
            <span class="col_sub">・未選択の場合は全従区分が対象</span><br />
            <span class="col_sub">・部分一致</span><br />
            <span class="col_sub">・複数の従区分名を入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">&nbsp;&nbsp;&nbsp;&nbsp;例：職員,役員</span><br />
            <span class="col_sub">・300文字以内</span>
          </td>
          <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td><span class="s1">以下の従区分名を</span></td></tr>
              <tr><td>
                <input type="radio" id="rdoJuKubun01" name="rdoJuKubun" value="1"  /><label for="rdoJuKubun01">抽出する</label>
                <input type="radio" id="rdoJuKubun02" name="rdoJuKubun" value="2"  /><label for="rdoJuKubun02">除外する</label>
              </td></tr>
              <tr>
                <td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.JuKubun);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td>
              </tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="sentaku">
            <textarea name="txtJuKubun" rows="2" cols="20" id="txtJuKubun" style="height:50px;width:95%;overflow:hidden;ime-mode:active;"></textarea>
          </td>
        </tr>
        <tr>
          <td class="koumoku2">
            <span class="col">職種</span><br />
            <span class="col_sub">・未選択の場合は全職種が対象</span>
          </td>
          <td>
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td>
                <input type="checkbox" id="chkSyoku_001" name="chkSyoku_001" value="1"  /><label for="chkSyoku_001">建築</label>
                <input type="checkbox" id="chkSyoku_002" name="chkSyoku_002" value="1"  /><label for="chkSyoku_002">土木</label>
                <input type="checkbox" id="chkSyoku_003" name="chkSyoku_003" value="1"  /><label for="chkSyoku_003">事務</label>
              </td></tr>
              <tr><td>
                <input type="checkbox" id="chkSyoku_004" name="chkSyoku_004" value="1"  /><label for="chkSyoku_004">設備</label>
                <input type="checkbox" id="chkSyoku_005" name="chkSyoku_005" value="1"  /><label for="chkSyoku_005">機電</label>
                <input type="checkbox" id="chkSyoku_006" name="chkSyoku_006" value="1"  /><label for="chkSyoku_006">その他</label>
              </td></tr>
              <tr>
                <td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.Syoku);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td>
              </tr>
            </table>
          </td>
        </tr>
      </table>
    </td>
  </tr>
</table>
</div>

<input type="hidden" id="hidKojinCode" name="hidKojinCode" value="" />
<input type="hidden" id="hidHonken" name="hidHonken" value="" />
<input type="hidden" id="hidKojinName" name="hidKojinName" value="" />
<input type="hidden" id="hidKojinKana" name="hidKojinKana" value="" />
<input type="hidden" id="hidRank" name="hidRank" value="" />
<input type="hidden" id="hidMailAddress" name="hidMailAddress" value="" />
<input type="hidden" id="hidYaku" name="hidYaku" value="" />
<input type="hidden" id="hidTenCode" name="hidTenCode" value="" />
<input type="hidden" id="hidJukubun" name="hidJukubun" value="" />
<input type="hidden" id="hidTenMei" name="hidTenMei" value="" />
<input type="hidden" id="hidJukubunName" name="hidJukubunName" value="" />
<input type="hidden" id="hidSosikiCode" name="hidSosikiCode" value="" />
<input type="hidden" id="hidSyoku" name="hidSyoku" value="" />
<input type="hidden" id="hidSosikiMei" name="hidSosikiMei" value="" />
<input type="hidden" id="hidNaisen" name="hidNaisen" value="" />
<input type="hidden" id="hidJGKubun" name="hidJGKubun" value="" />
<input type="hidden" id="hidGaisen" name="hidGaisen" value="" />
<input type="hidden" id="hidKoujiKubun" name="hidKoujiKubun" value="" />
<input type="hidden" id="hidWorksite" name="hidWorksite" value="" />
</form>


<form method="POST" id="form2" name="form2" action="./ADDR201.aspx" target="_top">
<input type="hidden" id="txtDummy" name="txtDummy" value="ADDR001" />
</form>

</body>
</html>

This is ADDR002.aspx source code from developer tool
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
<head>
  <title>メールアドレス検索</title>
<script type="text/javascript" language="javascript" src="../scripts/addr_common.js"></script>
<script type="text/javascript" language="javascript">
    function onClear() {
        $("chkKojinCode").checked = false;
        $("chkHonken").checked = false;
        $("chkKojinName").checked = false;
        $("chkKojinKana").checked = false;
        $("chkRank").checked = false;
        $("chkMailAddress").checked = false;
        $("chkYaku").checked = false;
        $("chkTenCode").checked = false;
        $("chkJukubun").checked = false;
        $("chkTenMei").checked = false;
        $("chkJukubunName").checked = false;
        $("chkSosikiCode").checked = false;
        $("chkSyoku").checked = false;
        $("chkSosikiMei").checked = false;
        $("chkNaisen").checked = false;
        $("chkJGKubun").checked = false;
        $("chkGaisen").checked = false;
        $("chkKoujiKubun").checked = false;
        $("chkWorksite").checked = false;
        return false;
    }
</script>
<style type="text/css">
  td.l { border-right:1px solid #000000;border-bottom:1px solid #000000; }
  td.l2 { border-right:1px solid #000000; }
  td.r { border-bottom:1px solid #000000; }
</style>
</head>
<body style="font-size:12px;">

<form method="POST" id="form1" name="form1">
<div style="width:95%;">
<table style="border:1px solid #000000;width:100%;" cellspacing="0" >
  <colgroup>
    <col width="50%" />
    <col width="50%" />
  </colgroup>
  <tr><th colspan="2" style="border-bottom:1px solid #000000;background-color:#ffffff;color:#008080;">出力項目</th></tr>
  <tr>
    <td class="l"><input type="checkbox" id="chkKojinCode" name="chkKojinCode" tabindex="1"  /><label for="chkKojinCode">個人コード</label></td>
    <td class="r"><input type="checkbox" id="chkHonken" name="chkHonken" tabindex="11"  /><label for="chkHonken">本務・兼務・他在籍・仮想在籍</label></td>
  </tr>
  <tr>
    <td class="l"><input type="checkbox" id="chkKojinName" name="chkKojinName" tabindex="2"  /><label for="chkKojinName">氏名</label></td>
    <td class="r"><input type="checkbox" id="chkRank" name="chkRank" tabindex="12"  /><label for="chkRank">職位（セキュリティランク）</label></td>
  </tr>
  <tr>
    <td class="l"><input type="checkbox" id="chkKojinKana" name="chkKojinKana" tabindex="3"  /><label for="chkKojinKana">姓名カナ</label></td>
    <td class="r"><input type="checkbox" id="chkYaku" name="chkYaku" tabindex="13"  /><label for="chkYaku">役職</label></td>

  </tr>
  <tr>
    <td class="l"><input type="checkbox" id="chkMailAddress" name="chkMailAddress" tabindex="4"  /><label for="chkMailAddress">メールアドレス</label></td>
    <td class="r"><input type="checkbox" id="chkJukubun" name="chkJukubun" tabindex="14"  /><label for="chkJukubun">従区分コード</label></td>
  </tr>
  <tr>
    <td class="l"><input type="checkbox" id="chkTenCode" name="chkTenCode" tabindex="5"  /><label for="chkTenCode">店コード</label></td>
    <td class="r"><input type="checkbox" id="chkJukubunName" name="chkJukubunName" tabindex="15"  /><label for="chkJukubunName">従区分名</label></td>
  </tr>
  <tr>
    <td class="l"><input type="checkbox" id="chkTenMei" name="chkTenMei" tabindex="6"  /><label for="chkTenMei">店名</label></td>
    <td class="r"><input type="checkbox" id="chkSyoku" name="chkSyoku" tabindex="16"  /><label for="chkSyoku">職種</label></td>
  </tr>
  <tr>
    <td class="l"><input type="checkbox" id="chkSosikiCode" name="chkSosikiCode" tabindex="7"  /><label for="chkSosikiCode">組織コード</label></td>
    <td class="r"><input type="checkbox" id="chkNaisen" name="chkNaisen" tabindex="17"  /><label for="chkNaisen">内線</label></td>
  </tr>
  <tr>
    <td class="l"><input type="checkbox" id="chkSosikiMei" name="chkSosikiMei" tabindex="8"  /><label for="chkSosikiMei">組織名</label></td>
    <td class="r"><input type="checkbox" id="chkGaisen" name="chkGaisen" tabindex="18"  /><label for="chkGaisen">外線</label></td>
  </tr>
  <tr>
    <td class="l"><input type="checkbox" id="chkJGKubun" name="chkJGKubun" tabindex="9"  /><label for="chkJGKubun">常設・現場</label></td>
    <td class="r"><input type="checkbox" id="chkWorksite" name="chkWorksite" tabindex="19"  /><label for="chkWorksite">勤務地</label></td>
  </tr>
  <tr>
    <td class="l2"><input type="checkbox" id="chkKoujiKubun" name="chkKoujiKubun" tabindex="10"  /><label for="chkKoujiKubun">工事区分</label></td>
    <td></td>
  </tr>
</table>
<table style="width:100%;" cellspacing="0" >
  <colgroup>
    <col width="50%" />
    <col width="50%" />
  </colgroup>
  <tr>
    <td></td>
    <td style="text-align:right"><a href="javascript:void(0);" onclick="return onClear();">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td>
  </tr>
</table>
</div>
</form>

</body>
</html>

this is ADDR003.aspx source code from developer tool

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
<head>
  <title>メールアドレス検索</title>
</head>
<body style="font-size:12px;">

<div>
<table>
  <tr><th style="color:#0070C0;font-weight:bold;font-size:14px;text-align:left;">検索条件を指定後、検索ボタンをクリックしてください。</th></tr>
  <tr>
    <td>
      <input type="button" id="btnSearch" name="btnSearch" value="検索" onclick="top.jyoken.DoSearch();" style="background-color:#ffff00;" />
    </td>
  </tr>
</table>   
</div>

</body>
</html>

1. from the  ADDR001.aspx
    check 現場、国内土木、国内建築
2. from the ADDR002.aspx
    check 個人コード、氏名
3. from the ADDR003.aspx
    click 検索 button
4. after click  検索 button, go ADDR101.aspx
5. payload is :
txtSosikiCode
txtSosikiName
chkJG_002
1
chkKKubun_001
1
chkKKubun_003
1
txtKojinCode
txtKyoryoku
ddtSecRankS
ddtSecRankE
txtYaku
txtJuKubun
hidKojinCode
1
hidHonken
hidKojinName
hidKojinKana
hidRank
hidMailAddress
hidYaku
hidTenCode
hidJukubun
hidTenMei
hidJukubunName
hidSosikiCode
hidSyoku
hidSosikiMei
hidNaisen
hidJGKubun
hidGaisen
hidKoujiKubun
hidWorksite

6. header is : 
Request URL
http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR101.aspx
Request method
POST
Status code
200 OK
Remote address
127.0.0.1:9000
Referrer policy
strict-origin-when-cross-origin
cache-control
private
connection
close
content-length
6931
content-type
text/html; charset=utf-8
date
Tue, 20 May 2025 01:59:45 GMT
server
Microsoft-IIS/10.0
x-aspnet-version
2.0.50727
x-powered-by
ASP.NET
accept
text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
accept-encoding
gzip, deflate
accept-language
ja,en-US;q=0.9,en;q=0.8
cache-control
no-cache
content-length
387
content-type
application/x-www-form-urlencoded
cookie
_ga=GA1.3.1806337504.1747208801; _ga_54DX7MV5YR=GS2.1.s1747214293$o2$g0$t1747214293$j60$l0$h0; amlbcookie=01; iPlanetDirectoryPro=AQIC5wM2LY4SfcxVOtB04BXH-vplR52rZinBXrcw8kAtNEw.*AAJTSQACMDMAAlNLABM0NzYxNzY3NjgyNTkyMzIzNjYyAAJTMQACMDE.*; ID=U53439; userID=53439; secCode=7; staffCode=0; yakCode=999; jyuCode=50; syoCode=09332; multiCode=0130; RedirectID=11010; TAMURL=http://www.fc.obayashi.co.jp/; ASP.NET_SessionId=lpa2o54535h1qe55i2yhaw55
host
it-nw.isc.obayashi.co.jp
origin
http://it-nw.isc.obayashi.co.jp
pragma
no-cache
proxy-connection
keep-alive
referer
http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR001.aspx
upgrade-insecure-requests
1
user-agent
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36

7. ADDR101.aspx
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title>メールアドレス・電話番号　検索</title>
<script type="text/javascript" language="javascript">
    function onDoSearch() {
        document.form1.submit();
    }
    function onCancel() {
        location.href = "ADDR003.aspx";
    }
</script>
<style type="text/css">
  table { font-size:12px; }
  #jyouken { background-color:#333366;width:100%; }
  #jyouken th { text-align:left;border:none;background-color:#ccccff;font-weight:normal;vertical-align:top; }
  #jyouken th.output { background-color:#666666;color:#ffffff; }
  #jyouken td { border:none;background-color:#dddddd;vertical-align:top; }
  .s1 { color:navy;font-size:13px; }
</style>
</head>
<body>
<div>

以下の条件で検索しますがよろしいですか。<br />

<table id="jyouken" cellpadding="0" cellspacing="1">
  <colgroup>
    <col style="width:40%" />
    <col style="width:60%" />
  </colgroup>
  <tr><th>支店</th><td style="word-break:break-all;"></td></tr>
  <tr><th>組織コード</th><td style="word-break:break-all;"></td></tr>
  <tr><th>組織名</th><td style="word-break:break-all;"></td></tr>
  <tr><th>常設・現場</th><td style="word-break:break-all;">現場</td></tr>
  <tr><th>工事区分</th><td style="word-break:break-all;">国内土木,国内建築</td></tr>
  <tr><th>個人コード</th><td style="word-break:break-all;"></td></tr>
  <tr>
    <th>協力スタッフ</th>
    <td style="word-break:break-all;">

    </td>
  </tr>
  <tr><th>本務・兼務・他在籍・仮想在籍</th><td style="word-break:break-all;"></td></tr>
  <tr><th>職位（セキュリティランク）</th><td style="word-break:break-all;"> ～ </td></tr>
  <tr>
    <th>役職</th>
    <td style="word-break:break-all;">

    </td>
  </tr>
  <tr>
    <th>従区分名</th>
    <td style="word-break:break-all;">

    </td>
  </tr>
  <tr><th>職種</th><td style="word-break:break-all;"></td></tr>
  <tr><th class="output">出力項目</th><td style="word-break:break-all;">個人コード</td></tr>
</table>

<input type="button" value="検索実行" onclick="onDoSearch();" />　
<input type="button" value="いいえ" onclick="onCancel();" />
</div>

<form method="POST" id="form1" name="form1" action="ADDR102.aspx" target="_self">
<input type="hidden" id="chkTen_09" name="chkTen_09" value="0" />
<input type="hidden" id="chkTen_11" name="chkTen_11" value="0" />
<input type="hidden" id="chkTen_15" name="chkTen_15" value="0" />
<input type="hidden" id="chkTen_10" name="chkTen_10" value="0" />
<input type="hidden" id="chkTen_26" name="chkTen_26" value="0" />
<input type="hidden" id="chkTen_12" name="chkTen_12" value="0" />
<input type="hidden" id="chkTen_13" name="chkTen_13" value="0" />
<input type="hidden" id="chkTen_14" name="chkTen_14" value="0" />
<input type="hidden" id="chkTen_16" name="chkTen_16" value="0" />
<input type="hidden" id="chkTen_17" name="chkTen_17" value="0" />
<input type="hidden" id="chkTen_19" name="chkTen_19" value="0" />
<input type="hidden" id="chkTen_27" name="chkTen_27" value="0" />

<input type="hidden" id="chkTen_31" name="chkTen_31" value="0" />
<input type="hidden" id="chkTen_32" name="chkTen_32" value="0">
<input type="hidden" id="chkTen_46" name="chkTen_46" value="0" />
<input type="hidden" id="chkTen_36" name="chkTen_36" value="0" />
<input type="hidden" id="chkTen_66" name="chkTen_66" value="0" />
<input type="hidden" id="txtSosikiCode" name="txtSosikiCode" value="" />
<input type="hidden" id="txtSosikiName" name="txtSosikiName" value="" />
<input type="hidden" id="chkJG_001" name="chkJG_001" value="0" />
<input type="hidden" id="chkJG_002" name="chkJG_002" value="1" />
<input type="hidden" id="chkKKubun_001" name="chkKKubun_001" value="1" />
<input type="hidden" id="chkKKubun_002" name="chkKKubun_002" value="0" />
<input type="hidden" id="chkKKubun_003" name="chkKKubun_003" value="1" />
<input type="hidden" id="chkKKubun_004" name="chkKKubun_004" value="0" />
<input type="hidden" id="txtKojinCode" name="txtKojinCode" value="" />
<input type="hidden" id="rdoKyoryoku" name="rdoKyoryoku" value="0" />
<input type="hidden" id="txtKyoryoku" name="txtKyoryoku" value="" />
<input type="hidden" id="chkHonKen_001" name="chkHonKen_001" value="0" />
<input type="hidden" id="chkHonKen_002" name="chkHonKen_002" value="0" />
<input type="hidden" id="chkHonKen_003" name="chkHonKen_003" value="0" />
<input type="hidden" id="chkHonKen_004" name="chkHonKen_004" value="0" />
<input type="hidden" id="ddtSecRankS" name="ddtSecRankS" value="" />
<input type="hidden" id="ddtSecRankE" name="ddtSecRankE" value="" />
<input type="hidden" id="rdoYaku" name="rdoYaku" value="0" />
<input type="hidden" id="txtYaku" name="txtYaku" value="" />
<input type="hidden" id="rdoJuKubun" name="rdoJuKubun" value="0" />
<input type="hidden" id="txtJuKubun" name="txtJuKubun" value="" />
<input type="hidden" id="chkSyoku_001" name="chkSyoku_001" value="0" />
<input type="hidden" id="chkSyoku_002" name="chkSyoku_002" value="0" />
<input type="hidden" id="chkSyoku_003" name="chkSyoku_003" value="0" />
<input type="hidden" id="chkSyoku_004" name="chkSyoku_004" value="0" />
<input type="hidden" id="chkSyoku_005" name="chkSyoku_005" value="0" />
<input type="hidden" id="chkSyoku_006" name="chkSyoku_006" value="0" />

<input type="hidden" id="hidKojinCode" name="hidKojinCode" value="1" />
<input type="hidden" id="hidHonken" name="hidHonken" value="0" />
<input type="hidden" id="hidKojinName" name="hidKojinName" value="0" />
<input type="hidden" id="hidKojinKana" name="hidKojinKana" value="0" />
<input type="hidden" id="hidRank" name="hidRank" value="0" />
<input type="hidden" id="hidMailAddress" name="hidMailAddress" value="0" />
<input type="hidden" id="hidYaku" name="hidYaku" value="0" />
<input type="hidden" id="hidTenCode" name="hidTenCode" value="0" />
<input type="hidden" id="hidJukubun" name="hidJukubun" value="0" />
<input type="hidden" id="hidTenMei" name="hidTenMei" value="0" />
<input type="hidden" id="hidJukubunName" name="hidJukubunName" value="0" />
<input type="hidden" id="hidSosikiCode" name="hidSosikiCode" value="0" />
<input type="hidden" id="hidSyoku" name="hidSyoku" value="0" />
<input type="hidden" id="hidSosikiMei" name="hidSosikiMei" value="0" />
<input type="hidden" id="hidNaisen" name="hidNaisen" value="0" />
<input type="hidden" id="hidJGKubun" name="hidJGKubun" value="0" />
<input type="hidden" id="hidGaisen" name="hidGaisen" value="0" />
<input type="hidden" id="hidKoujiKubun" name="hidKoujiKubun" value="0" />
<input type="hidden" id="hidWorksite" name="hidWorksite" value="0" />
</form>
</body>
</html>

8. when click 検索実行 button from ADDR001.aspx,
header : 
Request URL
http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR102.aspx
Request method
POST
Status code
200 OK
Remote address
127.0.0.1:9000
Referrer policy
strict-origin-when-cross-origin
cache-control
private
connection
close
content-length
6976
content-type
text/html; charset=utf-8
date
Tue, 20 May 2025 02:07:47 GMT
server
Microsoft-IIS/10.0
x-aspnet-version
2.0.50727
x-powered-by
ASP.NET
accept
text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
accept-encoding
gzip, deflate
accept-language
ja,en-US;q=0.9,en;q=0.8
cache-control
no-cache
content-length
844
content-type
application/x-www-form-urlencoded
cookie
_ga=GA1.3.1806337504.1747208801; _ga_54DX7MV5YR=GS2.1.s1747214293$o2$g0$t1747214293$j60$l0$h0; amlbcookie=01; iPlanetDirectoryPro=AQIC5wM2LY4SfcxVOtB04BXH-vplR52rZinBXrcw8kAtNEw.*AAJTSQACMDMAAlNLABM0NzYxNzY3NjgyNTkyMzIzNjYyAAJTMQACMDE.*; ID=U53439; userID=53439; secCode=7; staffCode=0; yakCode=999; jyuCode=50; syoCode=09332; multiCode=0130; RedirectID=11010; TAMURL=http://www.fc.obayashi.co.jp/; ASP.NET_SessionId=lpa2o54535h1qe55i2yhaw55
host
it-nw.isc.obayashi.co.jp
origin
http://it-nw.isc.obayashi.co.jp
pragma
no-cache
proxy-connection
keep-alive
referer
http://it-nw.isc.obayashi.co.jp/pick/frm/ADDR101.aspx
upgrade-insecure-requests
1
user-agent
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36

9. payload :
chkTen_09
0
chkTen_11
0
chkTen_15
0
chkTen_10
0
chkTen_26
0
chkTen_12
0
chkTen_13
0
chkTen_14
0
chkTen_16
0
chkTen_17
0
chkTen_19
0
chkTen_27
0
chkTen_31
0
chkTen_32
0
chkTen_46
0
chkTen_36
0
chkTen_66
0
txtSosikiCode
txtSosikiName
chkJG_001
0
chkJG_002
1
chkKKubun_001
1
chkKKubun_002
0
chkKKubun_003
1
chkKKubun_004
0
txtKojinCode
rdoKyoryoku
0
txtKyoryoku
chkHonKen_001
0
chkHonKen_002
0
chkHonKen_003
0
chkHonKen_004
0
ddtSecRankS
ddtSecRankE
rdoYaku
0
txtYaku
rdoJuKubun
0
txtJuKubun
chkSyoku_001
0
chkSyoku_002
0
chkSyoku_003
0
chkSyoku_004
0
chkSyoku_005
0
chkSyoku_006
0
hidKojinCode
1
hidHonken
0
hidKojinName
0
hidKojinKana
0
hidRank
0
hidMailAddress
0
hidYaku
0
hidTenCode
0
hidJukubun
0
hidTenMei
0
hidJukubunName
0
hidSosikiCode
0
hidSyoku
0
hidSosikiMei
0
hidNaisen
0
hidJGKubun
0
hidGaisen
0
hidKoujiKubun
0
hidWorksite
0

10. ADDR102.aspx

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title>メールアドレス・電話番号　検索</title>
<script>
    function onCSVDownLoad() {
        document.form1.submit();
        return false;
    }
</script>
<style type="text/css">
  table { font-size:12px; }
  #jyouken { background-color:#333366;width:100%; }
  #jyouken th { text-align:left;border:none;background-color:#ccccff;font-weight:normal;vertical-align:top; }
  #jyouken th.output { background-color:#666666;color:#ffffff; }
  #jyouken td { border:none;background-color:#dddddd;vertical-align:top; }
  .s1 { color:navy;font-size:13px; }
</style>
</head>
<body>
<div>


以下の条件で検索しますがよろしいですか。<br />

<table id="jyouken" cellpadding="0" cellspacing="1">
  <colgroup>
    <col style="width:40%" />
    <col style="width:60%" />
  </colgroup>
  <tr><th>支店</th><td style="word-break:break-all;"></td></tr>
  <tr><th>組織コード</th><td style="word-break:break-all;"></td></tr>
  <tr><th>組織名</th><td style="word-break:break-all;"></td></tr>
  <tr><th>常設・現場</th><td style="word-break:break-all;">現場</td></tr>
  <tr><th>工事区分</th><td style="word-break:break-all;">国内土木,国内建築</td></tr>
  <tr><th>個人コード</th><td style="word-break:break-all;"></td></tr>
  <tr>
    <th>協力スタッフ</th>
    <td style="word-break:break-all;">

    </td>
  </tr>
  <tr><th>本務・兼務・他在籍・仮想在籍</th><td style="word-break:break-all;"></td></tr>
  <tr><th>職位（セキュリティランク）</th><td style="word-break:break-all;"> ～ </td></tr>
  <tr>
    <th>役職</th>
    <td style="word-break:break-all;">

    </td>
  </tr>
  <tr>
    <th>従区分名</th>
    <td style="word-break:break-all;">

    </td>
  </tr>
  <tr><th>職種</th><td style="word-break:break-all;"></td></tr>
  <tr><th class="output">出力項目</th><td style="word-break:break-all;">個人コード</td></tr>
</table>
</div>

11530件　検索時間：0.141秒　<br />

<a href="javascript:void(0);" onclick="onCSVDownLoad();">CSV出力</a>

 （クリックするとダウンロードを開始します。）<br />
以上

<form method="POST" id="form1" name="form1" action="ADDR103.aspx" target="_blank">
<input type="hidden" id="chkTen_09" name="chkTen_09" value="0" />
<input type="hidden" id="chkTen_11" name="chkTen_11" value="0" />
<input type="hidden" id="chkTen_15" name="chkTen_15" value="0" />
<input type="hidden" id="chkTen_10" name="chkTen_10" value="0" />
<input type="hidden" id="chkTen_26" name="chkTen_26" value="0" />
<input type="hidden" id="chkTen_12" name="chkTen_12" value="0" />
<input type="hidden" id="chkTen_13" name="chkTen_13" value="0" />
<input type="hidden" id="chkTen_14" name="chkTen_14" value="0" />
<input type="hidden" id="chkTen_16" name="chkTen_16" value="0" />
<input type="hidden" id="chkTen_17" name="chkTen_17" value="0" />
<input type="hidden" id="chkTen_19" name="chkTen_19" value="0" />
<input type="hidden" id="chkTen_27" name="chkTen_27" value="0" />
<!--<input type="hidden" id="chkTen_30" name="chkTen_30" value="0" />-->
<input type="hidden" id="chkTen_31" name="chkTen_31" value="0" />
<input type="hidden" id="chkTen_32" name="chkTen_32" value="0">
<input type="hidden" id="chkTen_46" name="chkTen_46" value="0" />
<input type="hidden" id="chkTen_36" name="chkTen_36" value="0" />
<input type="hidden" id="chkTen_66" name="chkTen_66" value="0" />
<input type="hidden" id="txtSosikiCode" name="txtSosikiCode" value="" />
<input type="hidden" id="txtSosikiName" name="txtSosikiName" value="" />
<input type="hidden" id="chkJG_001" name="chkJG_001" value="0" />
<input type="hidden" id="chkJG_002" name="chkJG_002" value="1" />
<input type="hidden" id="chkKKubun_001" name="chkKKubun_001" value="1" />
<input type="hidden" id="chkKKubun_002" name="chkKKubun_002" value="0" />
<input type="hidden" id="chkKKubun_003" name="chkKKubun_003" value="1" />
<input type="hidden" id="chkKKubun_004" name="chkKKubun_004" value="0" />
<input type="hidden" id="txtKojinCode" name="txtKojinCode" value="" />
<input type="hidden" id="rdoKyoryoku" name="rdoKyoryoku" value="0" />
<input type="hidden" id="txtKyoryoku" name="txtKyoryoku" value="" />
<input type="hidden" id="chkHonKen_001" name="chkHonKen_001" value="0" />
<input type="hidden" id="chkHonKen_002" name="chkHonKen_002" value="0" />
<input type="hidden" id="chkHonKen_003" name="chkHonKen_003" value="0" />
<input type="hidden" id="chkHonKen_004" name="chkHonKen_004" value="0" />
<input type="hidden" id="ddtSecRankS" name="ddtSecRankS" value="" />
<input type="hidden" id="ddtSecRankE" name="ddtSecRankE" value="" />
<input type="hidden" id="rdoYaku" name="rdoYaku" value="0" />
<input type="hidden" id="txtYaku" name="txtYaku" value="" />
<input type="hidden" id="rdoJuKubun" name="rdoJuKubun" value="0" />
<input type="hidden" id="txtJuKubun" name="txtJuKubun" value="" />
<input type="hidden" id="chkSyoku_001" name="chkSyoku_001" value="0" />
<input type="hidden" id="chkSyoku_002" name="chkSyoku_002" value="0" />
<input type="hidden" id="chkSyoku_003" name="chkSyoku_003" value="0" />
<input type="hidden" id="chkSyoku_004" name="chkSyoku_004" value="0" />
<input type="hidden" id="chkSyoku_005" name="chkSyoku_005" value="0" />
<input type="hidden" id="chkSyoku_006" name="chkSyoku_006" value="0" />

<input type="hidden" id="hidKojinCode" name="hidKojinCode" value="1" />
<input type="hidden" id="hidHonken" name="hidHonken" value="0" />
<input type="hidden" id="hidKojinName" name="hidKojinName" value="0" />
<input type="hidden" id="hidKojinKana" name="hidKojinKana" value="0" />
<input type="hidden" id="hidRank" name="hidRank" value="0" />
<input type="hidden" id="hidMailAddress" name="hidMailAddress" value="0" />
<input type="hidden" id="hidYaku" name="hidYaku" value="0" />
<input type="hidden" id="hidTenCode" name="hidTenCode" value="0" />
<input type="hidden" id="hidJukubun" name="hidJukubun" value="0" />
<input type="hidden" id="hidTenMei" name="hidTenMei" value="0" />
<input type="hidden" id="hidJukubunName" name="hidJukubunName" value="0" />
<input type="hidden" id="hidSosikiCode" name="hidSosikiCode" value="0" />
<input type="hidden" id="hidSyoku" name="hidSyoku" value="0" />
<input type="hidden" id="hidSosikiMei" name="hidSosikiMei" value="0" />
<input type="hidden" id="hidNaisen" name="hidNaisen" value="0" />
<input type="hidden" id="hidJGKubun" name="hidJGKubun" value="0" />
<input type="hidden" id="hidGaisen" name="hidGaisen" value="0" />
<input type="hidden" id="hidKoujiKubun" name="hidKoujiKubun" value="0" />
<input type="hidden" id="hidWorksite" name="hidWorksite" value="0" />
</form>
</body>
</html>


11. when click 出力項目 link from ADDR102.aspx,
<script>
    function onCSVDownLoad() {
        document.form1.submit();
        return false;
    }
</script>
