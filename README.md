Okay, explaining this to a non-technical leader requires simplifying the core concepts. Here's how you can phrase it in English and Japanese, focusing on the main facts:

Key Facts to Convey:

The Problem: Our Azure application cannot find the login service (login.fc.obayashi.co.jp) on the internet.

Why it works locally: Your computer can find it because it uses a special, private "address book" (DNS) that's not available to the public internet (where our Azure app lives).

The Address: The login service has an "address" (100.64.1.12) that is like an internal company phone extension – not a public phone number.

The Solution (Concept): To fix this, we either need:

The login service to get a public "phone number" (public IP and DNS).

Or, our Azure application needs a secure, private "phone line" (VPN/ExpressRoute and internal DNS) to connect to the company's internal network.

Next Step: We need to talk to the IT team managing the login service to decide which solution is best and how to implement it.

English Explanation for a Non-Technical Leader:

"Hi [Leader's Name],

We've identified an issue where our new Azure application is unable to connect to the login.fc.obayashi.co.jp login service.

Essentially, our Azure application, which lives on the public internet, is trying to look up this login service in the public 'phone book' (public DNS), but it's finding that the address isn't listed there, or the listing says the address doesn't exist for public access.

You might have noticed that this login service works fine from your local computer. This is because your computer (or our office network) uses a special, private 'company phone book' (internal DNS). This private phone book has the correct internal 'extension number' (an IP address like 100.64.1.12) for the login service. This internal extension isn't something the public internet, and therefore our Azure application, can directly dial.

To solve this, we have a couple of main options that we need to discuss with the IT team responsible for the login service:

Make it Public: They could give the login service a proper public 'phone number' (a public IP address and public DNS record) so our Azure application can easily find and connect to it.

Create a Private Connection: We could set up a secure, private 'phone line' (like a VPN or a dedicated ExpressRoute connection) from our Azure application directly into the company's internal network. Our Azure app would then also use the company's internal 'phone book' to find the login service. This is more complex but might be necessary if the service must remain internal.

Our next step is to consult with the IT team that manages login.fc.obayashi.co.jp to determine the best way forward. This is more of a networking and infrastructure matter than a simple application bug.

Please let me know if you'd like more details."

Japanese Explanation for a Non-Technical Leader (日本語での説明):

「[リーダーの名前]様

Azureで開発中のアプリケーションが、ログインサービス（login.fc.obayashi.co.jp）に接続できない問題が判明しましたのでご報告します。

簡単に申しますと、私たちのAzureアプリケーションはインターネット上にあり、このログインサービスを公共の「電話帳」（パブリックDNS）で探そうとしています。しかし、その「電話帳」にはこのサービスのアドレスが載っていない、あるいは「このアドレスは公開されていません」と表示されてしまう状態です。

[リーダーの名前]様のローカルPCからはこのログインサービスが問題なく動作することにお気づきかもしれません。これは、ローカルPCや社内ネットワークが、特別な社内専用の「電話帳」（内部DNS）を使用しているためです。この社内「電話帳」には、ログインサービスの正しい社内「内線番号」（100.64.1.12のようなIPアドレス）が記録されています。この「内線番号」は、インターネット上のAzureアプリケーションからは直接ダイヤルできないものとお考えください。

この問題を解決するためには、主に2つの選択肢があり、ログインサービスを管理しているITチームと相談する必要があります。

サービスを公開する: ログインサービスに正式な公共の「電話番号」（公開IPアドレスと公開DNSレコード）を割り当ててもらう方法です。これにより、私たちのAzureアプリケーションが簡単に見つけて接続できるようになります。

プライベートな接続を確立する: 私たちのAzureアプリケーションから社内のネットワークへ、安全な専用の「電話回線」（VPNや専用線であるExpressRouteなど）を設ける方法です。そして、Azureアプリケーションも社内の「電話帳」を使ってログインサービスを探せるようにします。これは少し複雑になりますが、サービスを社内向けに限定する必要がある場合には有効な手段です。

次のステップとして、login.fc.obayashi.co.jpを管理しているITチームに相談し、最善の解決策を決定したいと考えております。これは単純なアプリケーションの不具合というよりは、ネットワークやインフラに関わる問題です。

ご不明な点がございましたら、お気軽にお尋ねください。」

Key phrases and their simpler analogies:

DNS (Domain Name System): "Address book" or "Phone book" (電話帳 - denwachou)

IP Address: "Address" or "Phone number" / "Extension number" (住所 - juusho / 電話番号 - denwa bangou / 内線番号 - naisen bangou)

Public DNS / Public IP: Accessible by anyone on the internet. (公共の - koukyou no)

Internal DNS / Private IP / CGNAT IP: Only accessible from within a specific private network. (社内の - shanai no / 内部の - naibu no)

Azure App Service: Our application "living on the public internet." (インターネット上に存在する - internet jou ni sonzai suru)

VPN / ExpressRoute: "Secure, private phone line" or "Dedicated private connection." (安全な専用線 - anzen na senyousen)

This should help your leader understand the core issue and why IT team collaboration is essential.




This nslookup output from your local machine is extremely revealing and confirms Reason 1: Split-Brain DNS / Internal DNS Resolution.

Let's break down what this output means:

サーバー: mobileap.qualcomm.com
Address: 192.168.0.1

This indicates your local machine is using a DNS server at the IP address 192.168.0.1.

192.168.0.1 is a private IP address, typically used for local network routers (like your home Wi-Fi router or a mobile hotspot's router functionality).

The hostname mobileap.qualcomm.com associated with it suggests this might be related to a mobile access point or tethering, or just how that router identifies itself.

Crucially, this is not a public internet DNS server.

DNS request timed out. (repeated twice)

This means your local DNS server (192.168.0.1) initially struggled or failed to resolve the name directly and might have had to forward the request or look it up in a specific way. This is common if the local DNS server has specific forwarding rules for certain domains or relies on upstream DNS servers that might be slow or have specific configurations.

権限のない回答: (Non-authoritative answer:)

This means the DNS server (192.168.0.1) that gave you the answer is not the primary or authoritative DNS server for the fc.obayashi.co.jp domain. It likely got the information from another DNS server it's configured to use or has cached.

名前: kite00.fc.obayashi.co.jp
Address: 100.64.1.12
Aliases: login.fc.obayashi.co.jp

This is the core of the information:

login.fc.obayashi.co.jp is an alias (CNAME record) for the actual hostname kite00.fc.obayashi.co.jp.

The hostname kite00.fc.obayashi.co.jp resolves to the IP address 100.64.1.12.

The IP Address 100.64.1.12:

This IP address falls within the 100.64.0.0/10 range (from 100.64.0.0 to 100.127.255.255).

This address block is reserved for Carrier-Grade NAT (CGNAT). It's used by ISPs and mobile carriers to share a limited number of public IPv4 addresses among many customers.

While technically not a "private" RFC 1918 address (like 10.x.x.x, 192.168.x.x), IPs in the CGNAT range are not generally routable or directly accessible from the public internet. They are used within the carrier's network.

This strongly implies that login.fc.obayashi.co.jp (via kite00.fc.obayashi.co.jp) is intended to be accessed from within a specific network environment (e.g., Obayashi's corporate network, or a network provided by a specific carrier that has a route to this CGNAT IP).

Conclusion from this nslookup:

The hostname login.fc.obayashi.co.jp is not resolving to a standard public IP address that would be reachable from your Azure App Service (which is on the public internet).

Your local machine can resolve it because your local DNS server (192.168.0.1) has access to DNS information (possibly provided by your company's network or a specific ISP/carrier configuration) that maps it to an IP (100.64.1.12) within a CGNAT or private-like address space.

Azure's public DNS resolver (168.63.129.16) correctly reports "Non-existent domain" (or would report no A/CNAME record publicly) because this mapping is not available in the global public DNS system.

How to Fix (for your Azure App Service):

Since your Azure App Service needs to reach this, and it appears to be an internal or specially networked resource, you cannot rely on public DNS resolution.

You need one of the solutions for connecting Azure to a private/internal network:

Confirm the Nature of login.fc.obayashi.co.jp with Obayashi IT:

Present this nslookup finding to the IT team responsible for login.fc.obayashi.co.jp.

Ask them:

Is this service intended to be accessible only from within the Obayashi corporate network or a specific partner network?

What is the recommended way for an externally hosted application (like your Azure App Service) to securely access this login service?

If Obayashi IT confirms it's internal and provides a way to connect (e.g., VPN/ExpressRoute):

Azure VNet Integration: You'll need to integrate your Azure App Service with an Azure Virtual Network (VNet).

Site-to-Site VPN or ExpressRoute: This VNet will need to be connected to Obayashi's on-premises network (where 100.64.1.12 is reachable) via a Site-to-Site VPN Gateway or an ExpressRoute circuit. This is a task for network administrators.

Custom DNS in Azure VNet: The Azure VNet must be configured to use Obayashi's internal DNS servers. These DNS servers must be reachable from the Azure VNet over the VPN/ExpressRoute. These internal DNS servers will be the ones that can resolve login.fc.obayashi.co.jp to 100.64.1.12.

Route All (likely needed): For your App Service to send DNS queries for login.fc.obayashi.co.jp to the internal DNS servers, you'll likely need to enable "Route All" outbound traffic from the App Service through the VNet.

Network Security Groups (NSGs) / Firewalls: Ensure that NSGs on the App Service's delegated subnet and any firewalls (Azure Firewall or on-premises) allow traffic from your App Service's outbound IPs to 100.64.1.12 on the required ports (HTTP/80, HTTPS/443).

If Obayashi IT can make login.fc.obayashi.co.jp publicly accessible (less likely given the CGNAT IP):

They would need to update their public DNS records to point login.fc.obayashi.co.jp (or a new, public hostname) to a globally routable public IP address where the service is hosted.

The service at that public IP would need to be configured to accept connections from the internet (or specifically from Azure IP ranges).

This is now primarily a networking and infrastructure challenge, not just a simple code or Azure App Service configuration fix. You need to collaborate with the team that owns and manages the login.fc.obayashi.co.jp service and its network environment.

Your immediate next step should be to share these findings with the relevant Obayashi IT personnel. Your Azure App Service is behaving correctly by not finding a publicly listed domain for login.fc.obayashi.co.jp.





nslookup login.fc.obayashi.co.jp
サーバー:  mobileap.qualcomm.com
Address:  192.168.0.1

DNS request timed out.
    timeout was 2 seconds.
DNS request timed out.
    timeout was 2 seconds.
権限のない回答:
名前:    kite00.fc.obayashi.co.jp
Address:  100.64.1.12
Aliases:  login.fc.obayashi.co.jp


C:\home>nameresolver.exe login.fc.obayashi.co.jp
Server: 168.63.129.16

*** 168.63.129.16 can't find login.fc.obayashi.co.jp: Non-existent domain

composer require symfony/dom-crawler

API call received for report type: org_info  
[2025-05-29 11:50:01] local.ERROR: Guzzle error during login: cURL error 6: Could not resolve host: login.fc.obayashi.co.jp (see https://curl.haxx.se/libcurl/c/libcurl-errors.html) {"exception":"[object] (GuzzleHttp\\Exception\\ConnectException(code: 0): cURL error 6: Could not resolve host: login.fc.obayashi.co.jp (see https://curl.haxx.se/libcurl/c/libcurl-errors.html) at C:\\home\\site\\wwwroot\\ccc\\vendor\\guzzlehttp\\guzzle\\src\\Handler\\CurlFactory.php:200)
[stacktrace]


Action 'HTTP_1' failed: The provided 'Http' action URI 'http://localhost:8080/ccc/admin/scrape-email-data' is not valid. The URI must be a well formed absolute URI not referencing local host or UNC path.


   if ($request->header('X-API-KEY') !== $this->apiKey) {
        Log::warning('API call with invalid API Key. Expected: [' . $this->apiKey . '], Received: [' . $request->header('X-API-KEY') . ']'); // Modified Log
        return Response::json(['error' => 'Unauthorized: Invalid API Key'], 401);
    }



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
