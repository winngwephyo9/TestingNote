
            if (strpos($body, '<TITLE>データベースへの問い合わせ中</TITLE>') === false && $response->getStatusCode() === 200) {
                // Check if the "database querying" message is NOT present and we have a 200 OK
                dd('Login successful!', $body);
            } else {
                dd('Login failed.', $body);
            }
            
            
^ "Login failed."
^ """

<HTML>
<HEAD>
<TITLE>データベースへの問い合わせ中</TITLE>
<META HTTP-EQUIV="Content-Type" Content="text/html; charset=x-sjis">

<script type="text/javascript" src="./jquery-1.10.2.min.js"></script>

<script language="JavaScript">

    //2021/12/08 S
    //建築ポータル起動有無設定読み込み
    var strDhoStatus = getCookie('dhostatus');
    //2021/12/08 E
    
\t//window.open("waitingSub.htm","subwin","resizable,toolbar=no,menubar=no,width=300,height=170,top=50,left=50");
    sai = getCookie('stopsystem').charAt(0);
    //承認小窓はCookie"stopsystem"の1桁目が"1"のときは表示しない
   \tif(sai != '1'){
   \t    //window.open("waitingSub.htm","subwin","resizable,toolbar=no,menubar=no,width=320,height=550,top=50,left=50");
        //2022/12/28 S
   \t    //window.open("waitingSub.htm","subwin","resizable=no,toolbar=no,menubar=no,width=320,height=650,top=30,left=20");
   \t    window.open("waitingSub.htm","subwin","resizable=no,toolbar=no,menubar=no,width=320,height=420,top=30,left=20");
        //2022/12/28 E
\t}
\t//土木一日一話 → 土木ポータル(2024/04/03)
\tsec = parseInt(getCookie('secCode'));
\tshokushu = right(getCookie('multiCode'),3);
\t//2023/03/17 S 特定の部署に本務所属するユーザも起動可能となるよう改修
\tvar strSyoCode = getCookie('syoCode');

\t//if(sec <= 11 && (shokushu == '110' || shokushu == '111' || shokushu == '140' || shokushu == '141')){
\tif((sec <= 11 && (shokushu == '110' || shokushu == '111' || shokushu == '140' || shokushu == '141')) || (strSyoCode == '09231' || strSyoCode == '09232' || str ▶
\t    //2023/03/17 E

        //2019/06/12 起動URL、起動設定変更S
\t    //window.open("http://www.dho.cvl.obayashi.co.jp/civil_portal01/ichinichi_ichiwa/skk_ad.html","cvlskk", "width=500,height=570,top=30,left=400,scrollbars=n ▶
\t    //window.open('http://www.dho.cvl.obayashi.co.jp/new_ichinichi_ichiwa/','cvlskk','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,t ▶
\t    //2019/06/12 起動URL、起動設定変更 E

      //2024/04/03 土木ポータル変更S
        //左位置
        var subx = 400;
        //上位置
        var suby = 30;        
        //幅
        var subw = 1300;
        //高さ
        var subh = 800;
                    
        //画面幅取得
        var intWinW = window.screen.width;

        if (intWinW < 1900){
            //画面サイズが1900未満の場合
            subw = 910;
            subh = 600;
        }

        window.open("http://www.dho.cvl.obayashi.co.jp/c-portal/", "cvlskk", "width=" + subw + ",height=" + subh + ",top=" + suby + ",left=" + subx + ",scrollba ▶

      //2024/04/03 土木ポータル変更E

\t}
\t//生産性５％アップ
\tccc = '';
\tsec = parseInt(getCookie('secCode'));
\tjyu = parseInt(getCookie('jyuCode'),10);
\tshokushu = right(getCookie('multiCode'),3);
\tuid = getCookie('userID');
    var jg = getCookie('multiCode').substring(0,1);
    //2021/10/18 S 起動廃止
    //if(uid != '68199' && sec <= 11 && (jyu >= 5 && jyu <= 74) && ((shokushu == '100' && ccc == '3')||(shokushu == '101' && ccc == '3')|| shokushu == '130' ||  ▶
    //    window.open('http://www.dho.bc.obayashi.co.jp/new_seisan5up/','seisan5up','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,top= ▶
\t//}
    //2021/10/18 E

    //2021/10/18 S 新建築ポータル起動処理実装
    var strOpt = 'top=30,left=360,scrollbars=yes,resizable=yes,menubar=yes,location=yes,directories=yes,status=yes';
    //2023/02/22 S 建築ポータル起動条件変更対応
    //if(uid != '68199' && sec <= 11 && (jyu >= 5 && jyu <= 74) && ((shokushu == '100' && ccc == '3')||(shokushu == '101' && ccc == '3')|| shokushu == '130' ||  ▶
    if(uid != '68199' && sec <= 15 && ((jyu >= 5 && jyu <= 74) || jyu == 90 || jyu == 94 || jyu == 95) && ((shokushu == '100' && ccc == '3')||(shokushu == '101' ▶
        //2023/02/22 E
        //旧「できる！10%UP(さらに昔の生産性５％アップ)」起動条件と合致する場合は新建築ポータルを起動

        //2021/12/08 S
        //window.open('http://www.dho2.bc.obayashi.co.jp/','dho2',strOpt);
        //cookie値(dhostatus)が'1'でない場合、建築ポータルを起動。'1'の場合は起動しない
        if (strDhoStatus != '1'){window.open('http://www.dho2.bc.obayashi.co.jp/','dho2',strOpt);}
        //2021/12/08 E

\t} else {
        var blnChkTza01 = CheckTza01();
        if (blnChkTza01){
            //建築ポップアップの起動条件と合致する場合は新建築ポータルを起動

            //2021/12/08 S
            //window.open('http://www.dho2.bc.obayashi.co.jp/','dho2',strOpt);
            //cookie値(dhostatus)が'1'でない場合、建築ポータルを起動。'1'の場合は起動しない
            if (strDhoStatus != '1'){window.open('http://www.dho2.bc.obayashi.co.jp/','dho2',strOpt);}
            //2021/12/08 E
        }
    }
    //2021/10/18 E

\t//RNイントラPOPUP S
//2020/06/05 バイパスするよう設定
//    if (HttpRequest_Send("http://toraburun.fc.obayashi.co.jp/RNPAPI/API/RNPA001.aspx", null) == '1') {
    if (HttpRequest_Send("ReqTrans.aspx?p=http://toraburun.fc.obayashi.co.jp/RNPAPI/API/RNPA001.aspx", null) == '1') {
//2020/06/05
        //画面高さ取得
        var intWinH = window.screen.height;            
        //900を超えている場合は、高さ900とする。900未満の場合はタスクバー領域等を考慮し、少し縮めた高さに設定
        if (intWinH > 900) {intWinH = 900;}else{intWinH += -50;}
        //起動
        window.open("http://toraburun.fc.obayashi.co.jp/RNP/frm/RNPS001.aspx", "rnp", "width=576,height=" + intWinH + ",top=30,left=780,scrollbars=yes,resizable ▶
    }
\t//RNイントラPOPUP E

    //2023/12/05 トラブルＤＢ S
    if (HttpRequest_Send("ReqTrans.aspx?p=http://toraburun.fc.obayashi.co.jp/toradbapi/TDBPA001.aspx", null) == '1') {
        var subw = 632;
        var subh = 446;
        var subx = (window.screen.width - subw) / 2;
        var sbuy = (window.screen.height - subh) / 2;

        window.open("http://www.brc.osk.obayashi.co.jp/troubleDB/trial/troubldb_popup.html", "tdbp", "width=" + subw + ",height=" + subh + ",top=" + sbuy + ",le ▶

    }
    //2023/12/05 トラブルＤＢ E

    //2021/10/18 S 起動廃止
    //Exec_Tza01();
    //2021/10/18 S

\twindow.location.href = "http://www.fc.obayashi.co.jp";
\t
function getCookie(key){
    tmp=document.cookie+";";
    tmp1=tmp.indexOf(key, 0);
    if(tmp1!=-1){
        tmp=tmp.substring(tmp1, tmp.length);
        start=tmp.indexOf("=", 0)+ 1;
        end=tmp.indexOf(";", start);
        return(unescape(tmp.substring(start, end)));
    }
    return("");
}
//
// 文字列を右側から指定文字数だけ取り出す関数
//
function right( str, n ) {
    l = str.length;
    if (n>l) n=l;
    return( str.substring(l-n,l) );
}
//RNイントラPOPUP S
//通信処理
function HttpRequest_Send(strUrl, strParam) {
    var ajax = null;

    for ( i=0; i < 3; i++) {
        try {
            if (window.XMLHttpRequest) { ajax = new XMLHttpRequest(); }
            else if (window.ActiveXObject) {
                try {
                    ajax = new ActiveXObject("Msxml2.XMLHTTP");
                } catch (e) {
                    ajax = new ActiveXObject("Microsoft.XMLHTTP");
                }
            }
            //通信実行
            ajax.open("POST", strUrl, false);
            ajax.timeout = 5000;
            ajax.send(strParam);
            return ajax.responseText;

        } catch (e) {
            if(i == 2){
                return '0';
            }else{
                //alert(i + e.message);
                continue;
            }
        } finally {
            ajax = null;
        }
    }
}
//RNイントラPOPUP E

//2021/10/18 S 改修
//TZA01アクセス権限認証
function CheckTza01() {
    //参照先サーバ(★導入環境に合わせて設定変更)
    var strRootUrl = 'http://mieruka-popup.fc.obayashi.co.jp';

    //WebAPIに対するURL設定
    var strUrl = 'ReqTrans.aspx?p=' + strRootUrl + '/tza01/TZAA000/TZAA000';

    try {                
        //通信実行
        var objRet = sendQuery(strUrl, '');

        //結果確認(エラー、もしくはアクセス権なしの場合は処理抜ける)
        if (objRet == undefined || objRet == null) { return false; }

        var objRetJson = (new Function("return " + objRet))();

        if (objRetJson.ErrCode != 0 || objRetJson.RetCode != 1) { return false; }

        //認証ＯＫの場合はTrueを返却
        return true;

    } catch (e) {
        //処理を止めない為、スルー
        return false;
    }    
}
//2021/10/18 E

//通信(同期)
function sendQuery(send_url, send_data) {
    var ret_data;

    $.ajax({
        type: 'POST',
        async: false,
        url: send_url,
        data: send_data,
        success: function (data) {
            ret_data = data;
        }
    });
    return ret_data
}
//2020/05/28 E

</SCRIPT>
</HEAD>
<BODY>
<CENTER>
<H3>処理中です。しばらくお待ち下さい。</H3>
</CENTER>
</BODY>
</HTML>
"""


<?php

namespace App\Http\Controllers;

use GuzzleHttp\Client;
use Illuminate\Http\Request;

class ObayashiLoginController extends Controller
{
    public function login()
    {
        $client = new Client([
            'cookies' => true, // Enable cookie handling
            'headers' => [
                'User-Agent' => 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36',
                'Accept' => 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
                'Accept-Encoding' => 'gzip, deflate',
                'Accept-Language' => 'ja-JP,ja;q=0.9,en-US;q=0.8,en;q=0.7',
                'Cache-Control' => 'no-cache',
                'Connection' => 'keep-alive',
                'Origin' => 'http://login.fc.obayashi.co.jp',
                'Pragma' => 'no-cache',
                'Referer' => 'http://login.fc.obayashi.co.jp/sso/UI/Login?type=obayashi&goto=http%3A%2F%2Fintralogin.fc.obayashi.co.jp%2Fcgi-bin%2Fclogin.cgi%3Faccess%3Dhttp%253A%252F%252Fwww.fc.obayashi.co.jp%252F%252F',
                'Upgrade-Insecure-Requests' => '1',
            ],
        ]);

        $loginUrl = 'http://login.fc.obayashi.co.jp/sso/UI/Login';
        $successRedirectUrl = 'http://intralogin.fc.obayashi.co.jp/cgi-bin/clogin.cgi?access=http%3A%2F%2Fwww.fc.obayashi.co.jp%2F';

        $username = 'YOUR_USERNAME'; // Replace with the actual username
        $password = 'YOUR_PASSWORD'; // Replace with the actual password
        $groupCompanyCode = 'U'; // Replace with the desired group company code (e.g., 'U' for 大林組)

        try {
            // 1. Get the initial login page to potentially retrieve any necessary tokens or cookies
            $response = $client->get($loginUrl);
            $html = (string) $response->getBody();

            // You might need to parse the HTML here to extract any dynamic values
            // if the form has anti-CSRF tokens or other dynamic fields.
            // Based on the provided HTML, it doesn't seem to have any critical dynamic tokens.

            // 2. Prepare the login form data
            $loginData = [
                'IDToken0-0' => $groupCompanyCode,
                'IDToken0' => $groupCompanyCode, // Assuming the company code is the same as the group code initially
                'IDToken1' => $groupCompanyCode . $username, // Apply the JavaScript logic
                'IDToken2' => $password,
                'Login.Submit' => 'ログイン',
                'goto' => 'aHR0cDovL2ludHJhbG9naW4uZmMub2JheWFzaGkuY28uanAvY2dpLWJpbi9jbG9naW4uY2dpP2FjY2Vzcz1odHRwOi8vd3d3LmZjLm9iYXlhc2hpLmNvLmpwLw==',
                'gotoOnFail' => '',
                'SunQueryParamsString' => 'dHlwZT1vYmF5YXNoaQ==',
                'encoded' => 'true',
                'type' => 'obayashi',
                'gx_charset' => 'UTF-8',
                'IDButton' => 'ログイン', // Set the IDButton value as per the JavaScript
            ];

            // 3. Submit the login form
            $response = $client->post($loginUrl, [
                'form_params' => $loginData,
            ]);

            // 4. Check if the login was successful by inspecting the response and cookies
            // A successful login should result in a redirect to the success URL
            if ($response->getStatusCode() === 302 && $response->getHeaderLine('Location') === $successRedirectUrl) {
                // Optionally, you can make a subsequent request to the redirected URL
                $successPageResponse = $client->get($successRedirectUrl);
                $successPageContent = (string) $successPageResponse->getBody();

                // Do something with the successful login page content
                dd('Login successful!', $successPageContent);
            } else {
                // Login failed
                dd('Login failed.', (string) $response->getBody());
            }

        } catch (\GuzzleHttp\Exception\GuzzleException $e) {
            dd('An error occurred: ' . $e->getMessage());
        }
    }
}

![画像 (16)](https://github.com/user-attachments/assets/813622b4-a900-46cd-ab5d-ecc54c822c14)
![画像 (14)](https://github.com/user-attachments/assets/25ccd9e5-c8d6-4f79-8235-cbd264344471)
[login.txt](https://github.com/user-attachments/files/20218578/login.txt)
