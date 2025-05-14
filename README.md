// ✅ Check for a known element only present after login
try {
  await page.waitForSelector('a[href*="電話帳"]', { timeout: 5000 });
  console.log('✅ Login successful');
} catch (e) {
  console.error('❌ Login failed: expected element not found');
  await browser.close();
  process.exit(1); // Exit with error
}

<?php

namespace App\Http\Controllers;

use Goutte\Client;
use Illuminate\Http\Request;


class ScraperController extends Controller
{
    private $result = array();
    public function scraper()
    {
        $client = new Client();
        $loginUrl = 'http://login.fc.obayashi.co.jp/sso/UI/Login';
        $goto = 'aHR0cDovL2ludHJhbG9naW4uZmMub2JheWFzaGkuY28uanAvY2dpLWJpbi9jbG9naW4uY2dpP2FjY2Vzcz1odHRwOi8vd3d3LmZjLm9iYXlhc2hpLmNvLmpwLw==';
        $formParams = [
            'IDToken0-0' => '', // May be optional
            'IDToken0' => '',   // May be optional
            'IDToken1' => 'UHR757',
            'IDToken2' => 'Win22222',
            'IDButton' => 'ログイン',
            'Login.Submit' => 'ログイン',
        ];
        $loginResponse = $client->request('POST', $loginUrl, [
            'headers' => [
                'User-Agent' => 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)',
                'Accept' => 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
                'Referer' => $loginUrl . '?type=obayashi&goto=' . urlencode(base64_decode($goto)),
                'Origin' => 'http://login.fc.obayashi.co.jp',
                'Content-Type' => 'application/x-www-form-urlencoded',
            ],
            'form_params' => $formParams,
        ]);
        echo "<pre>";
        print_r($loginResponse);
    }
}


login Response is like that
Symfony\Component\DomCrawler\Crawler Object
(
    [uri:protected] => http://login.fc.obayashi.co.jp/sso/UI/Login
    [defaultNamespacePrefix:Symfony\Component\DomCrawler\Crawler:private] => default
    [namespaces:Symfony\Component\DomCrawler\Crawler:private] => Array
        (
        )

    [cachedNamespaces:Symfony\Component\DomCrawler\Crawler:private] => ArrayObject Object
        (
            [storage:ArrayObject:private] => Array
                (
                )

        )

    [baseHref:Symfony\Component\DomCrawler\Crawler:private] => http://login.fc.obayashi.co.jp/sso/UI/Login
    [document:Symfony\Component\DomCrawler\Crawler:private] => DOMDocument Object
        (
            [doctype] => (object value omitted)
            [implementation] => (object value omitted)
            [documentElement] => (object value omitted)
            [actualEncoding] => UTF-8
            [encoding] => UTF-8
            [xmlEncoding] => UTF-8
            [standalone] => 1
            [xmlStandalone] => 1
            [version] => 
            [xmlVersion] => 
            [strictErrorChecking] => 1
            [documentURI] => 
            [config] => 
            [formatOutput] => 
            [validateOnParse] => 1
            [resolveExternals] => 
            [preserveWhiteSpace] => 1
            [recover] => 
            [substituteEntities] => 
            [nodeName] => #document
            [nodeValue] => 
            [nodeType] => 13
            [parentNode] => 
            [childNodes] => (object value omitted)
            [firstChild] => (object value omitted)
            [lastChild] => (object value omitted)
            [previousSibling] => 
            [nextSibling] => 
            [attributes] => 
            [ownerDocument] => 
            [namespaceURI] => 
            [prefix] => 
            [localName] => 
            [baseURI] => 
            [textContent] => 
  大林組イントラネットログイン画面
        var defaultBtn = 'Submit';
        var elmCount = 0;
        /** submit form with default command button */
        function defaultSubmit() {
            LoginSubmit(defaultBtn);
        }
      /**
       * submit form with given button value
       *
       * @param value of button
       */
       
      function LoginSubmit(value) {
        if( !input_Check() ){
          return false;
        }
        document.getElementById("IDToken1").value =
        document.getElementById("IDToken0").value +
        document.getElementById("IDToken1").value;
        aggSubmit();
        var hiddenFrm = document.forms['Login'];
        if (hiddenFrm != null) {
            hiddenFrm.elements['IDButton'].value = value;
            if (this.submitted) {
                alert("The request is currently being processed");
            } else {
                this.submitted = true;
                document.getElementById("submit_button").disabled = true;
		hiddenFrm.submit();
            }
        }
      }
   
  // Empty script so IE5.0 Windows will draw table and button borders
  //
     function groupID_change(groupID){
        //グループ会社を選択したら、ウィンドウの会社コードを変更する
        document.getElementById("IDToken0").value = groupID; 
      }
      function input_Check(){
        var username = document.getElementById("IDToken1").value;
        var password = document.getElementById("IDToken2").value;
        var groupname = document.getElementById("IDToken0-0").value;
        var groupcode = document.getElementById("IDToken0").value;
        // groupID empty check (for Group)
        if ( "" == "group" ){
          if ( groupname.length == 0 || groupcode.length == 0 ){
            document.getElementById("b-message-empty").innerHTML = "グループ会社が選択されていません。";
            return false;
          }
        }
        // userid empty check
        if( username.length == 0 ){
          document.getElementById("b-message-empty").innerHTML = "ユーザＩＤが入力されていません。";
          return false;
        }
        // userid format check (for Obayashi)
         if ( "" == "obayashi" ){
          if ( username.length != 6 || !(username.match(/^U/gi )) ){
            if ( !(username.match(/^(A|B)/gi )) ){
              document.getElementById("b-message-empty").innerHTML = "ユーザＩＤの入力に誤りがあります。";
              return false;
            } else {
              if ( username.length < 5 || username.length > 15 ) {
                document.getElementById("b-message-empty").innerHTML = "ユーザＩＤの入力に誤りがあります。";
                return false;
              }
            }
          }
        } 
        // password empty check
        if( password.length == 0 ){
          document.getElementById("b-message-empty").innerHTML = "パスワードが入力されていません。";
          return false;
        }
        return true;
      }
      大林組イントラネットログイン画面
      ユーザIDとパスワードを半角で入力してください
     if (document.getElementById("IDToken1")) {
    if (document.getElementById("IDToken1").type == "password") {
      bgroupMiddle.setAttribute("class", "b-groupMiddle_only");
      bgroupMiddle.setAttribute("className", "b-groupMiddle_only");
    }
  }
  グループ会社
      大林組
      アトリエ・ジーアンドビー
      オーク情報システム
      オーク設備工業
      大林クリーンエナジー
      大林神栖バイオマス発電
      大月バイオマス発電
      大月ウッドサプライ
      大林新星和不動産
      大林デザインパートナーズ
      大林道路
      大林ファシリティーズ
      相馬環境サービス
      特研メカトロニクス
      内外テクノス
      ルポンドシエル
      ＰＬｉＢＯＴ
      大林リアルティマネジメント
      オプライゾン
      オークロード熊本
      富士防災
      サイプレス・スナダヤ
      オークロード高知
      オークロード広島
      ＭｉＴＡＳＵＮ
      
      築柴
      
  
  



  
    会社コード
    ユーザＩＤ
      if (document.getElementById("IDToken1")) {
    if (document.getElementById("IDToken1").type == "password") {
      bgroupMiddle.setAttribute("class", "b-groupMiddle_only");
      bgroupMiddle.setAttribute("className", "b-groupMiddle_only");
    }
  }




       
            
       











  
    
    パスワード
    
    
  


  if (document.getElementById("IDToken1")) {
    if (document.getElementById("IDToken1").type == "password") {
      bgroupMiddle.setAttribute("class", "b-groupMiddle_only");
      bgroupMiddle.setAttribute("className", "b-groupMiddle_only");
    }
  }













   
    
    
        
    
    



     
        パスワードに入力した文字を確認したい場合はここをクリックしてください
         
             
                 
             
             
         
      
      
      
               
     
    

    
      

    
      
        ユーザIDがロックされたり、パスワードを忘れた場合
          （Windowsログオンのパスワードを忘れた場合もこちら）
        パスワードの変更方法
         パスワードに関する質問
      
    

    
      パスワードの変更は即時(数分程度)に反映されます。
      ※イントラネットに掲載されている情報には機密情報（社外秘、関係者外秘）が含まれますので、取り扱いには充分ご注意ください。
    


    

    
      

    
      
        パスワードに関する質問
        ユーザIDがロックされたり、パスワードを忘れた場合
      
    

    
      パスワードの変更は即時(数分程度)に反映されます。
      ※グループポータルに掲載されている情報には機密情報（社外秘、関係者外秘）が含まれますので、取り扱いには充分ご注意ください。
    


    
　　
　　
     
     
      
      Copyright(C) 2016 Obayashi Corporation. All Rights Reserved..
      
      
       
        
         if (elmCount != null) {
          document.write("
");
         }
        
        
        
        
        
        
        
       
      
     


  
  
    
  



        )

    [nodes:Symfony\Component\DomCrawler\Crawler:private] => Array
        (
            [0] => DOMElement Object
                (
                    [tagName] => html
                    [schemaTypeInfo] => 
                    [nodeName] => html
                    [nodeValue] => 












  大林組イントラネットログイン画面

  
  
  
  
  
  
  
  
  
  
  
    
  
  
  
  
  
  
  

  
  
    
  
    
      
        var defaultBtn = 'Submit';
        var elmCount = 0;
        /** submit form with default command button */
        function defaultSubmit() {
            LoginSubmit(defaultBtn);
        }
      /**
       * submit form with given button value
       *
       * @param value of button
       */
       
      function LoginSubmit(value) {
        if( !input_Check() ){
          return false;
        }
        document.getElementById("IDToken1").value =
        document.getElementById("IDToken0").value +
        document.getElementById("IDToken1").value;
        aggSubmit();
        var hiddenFrm = document.forms['Login'];
        if (hiddenFrm != null) {
            hiddenFrm.elements['IDButton'].value = value;
            if (this.submitted) {
                alert("The request is currently being processed");
            } else {
                this.submitted = true;
                document.getElementById("submit_button").disabled = true;
		hiddenFrm.submit();
            }
        }
      }
      
    
  
  
  // Empty script so IE5.0 Windows will draw table and button borders
  //
  
  
      function groupID_change(groupID){
        //グループ会社を選択したら、ウィンドウの会社コードを変更する
        document.getElementById("IDToken0").value = groupID; 
      }
      function input_Check(){
        var username = document.getElementById("IDToken1").value;
        var password = document.getElementById("IDToken2").value;
        var groupname = document.getElementById("IDToken0-0").value;
        var groupcode = document.getElementById("IDToken0").value;
        // groupID empty check (for Group)
        if ( "" == "group" ){
          if ( groupname.length == 0 || groupcode.length == 0 ){
            document.getElementById("b-message-empty").innerHTML = "グループ会社が選択されていません。";
            return false;
          }
        }
        // userid empty check
        if( username.length == 0 ){
          document.getElementById("b-message-empty").innerHTML = "ユーザＩＤが入力されていません。";
          return false;
        }
        // userid format check (for Obayashi)
         if ( "" == "obayashi" ){
          if ( username.length != 6 || !(username.match(/^U/gi )) ){
            if ( !(username.match(/^(A|B)/gi )) ){
              document.getElementById("b-message-empty").innerHTML = "ユーザＩＤの入力に誤りがあります。";
              return false;
            } else {
              if ( username.length < 5 || username.length > 15 ) {
                document.getElementById("b-message-empty").innerHTML = "ユーザＩＤの入力に誤りがあります。";
                return false;
              }
            }
          }
        } 
        // password empty check
        if( password.length == 0 ){
          document.getElementById("b-message-empty").innerHTML = "パスワードが入力されていません。";
          return false;
        }
        return true;
      }
  




  
    大林組イントラネットログイン画面
  



 
  
  
  
  
   

    
    
      
        
      
      
      
    
    
    
    
    
      ユーザIDとパスワードを半角で入力してください
    
    
     
     
     
     
     
      
    

    
       
            
       












  if (document.getElementById("IDToken1")) {
    if (document.getElementById("IDToken1").type == "password") {
      bgroupMiddle.setAttribute("class", "b-groupMiddle_only");
      bgroupMiddle.setAttribute("className", "b-groupMiddle_only");
    }
  }




       
            
       




  
    グループ会社
    
      
      
      大林組
      
      アトリエ・ジーアンドビー
      
      オーク情報システム
      
      オーク設備工業
      
      大林クリーンエナジー
      
      大林神栖バイオマス発電
      
      大月バイオマス発電
      
      大月ウッドサプライ
      
      大林新星和不動産
      
      大林デザインパートナーズ
      
      大林道路
      
      大林ファシリティーズ
      
      相馬環境サービス
      
      特研メカトロニクス
      
      内外テクノス
      
      ルポンドシエル
      
      ＰＬｉＢＯＴ
      
      大林リアルティマネジメント
      
      オプライゾン
      
      オークロード熊本
      
      富士防災
      
      サイプレス・スナダヤ
      
      オークロード高知
      
      オークロード広島
      
      ＭｉＴＡＳＵＮ
      
      築柴
      会社コード
    ユーザＩＤ
   if (document.getElementById("IDToken1")) {
    if (document.getElementById("IDToken1").type == "password") {
      bgroupMiddle.setAttribute("class", "b-groupMiddle_only");
      bgroupMiddle.setAttribute("className", "b-groupMiddle_only");
    }
  }
 パスワード
  if (document.getElementById("IDToken1")) {
    if (document.getElementById("IDToken1").type == "password") {
      bgroupMiddle.setAttribute("class", "b-groupMiddle_only");
      bgroupMiddle.setAttribute("className", "b-groupMiddle_only");
    }
  }
パスワードに入力した文字を確認したい場合はここをクリックしてください
        ユーザIDがロックされたり、パスワードを忘れた場合
          （Windowsログオンのパスワードを忘れた場合もこちら）
        パスワードの変更方法
         パスワードに関する質問
パスワードの変更は即時(数分程度)に反映されます。
      ※イントラネットに掲載されている情報には機密情報（社外秘、関係者外秘）が含まれますので、取り扱いには充分ご注意ください。
パスワードに関する質問
        ユーザIDがロックされたり、パスワードを忘れた場合
      パスワードの変更は即時(数分程度)に反映されます。
      ※グループポータルに掲載されている情報には機密情報（社外秘、関係者外秘）が含まれますので、取り扱いには充分ご注意ください。
      Copyright(C) 2016 Obayashi Corporation. All Rights Reserved..
         if (elmCount != null) {
          document.write("
");
         }
          [nodeType] => 1
                    [parentNode] => (object value omitted)
                    [childNodes] => (object value omitted)
                    [firstChild] => (object value omitted)
                    [lastChild] => (object value omitted)
                    [previousSibling] => (object value omitted)
                    [nextSibling] => 
                    [attributes] => (object value omitted)
                    [ownerDocument] => (object value omitted)
                    [namespaceURI] => 
                    [prefix] => 
                    [localName] => html
                    [baseURI] => 
                    [textContent] => 
 大林組イントラネットログイン画面
 var defaultBtn = 'Submit';
        var elmCount = 0;
        /** submit form with default command button */
        function defaultSubmit() {
            LoginSubmit(defaultBtn);
        }
      /**
       * submit form with given button value
       *
       * @param value of button
       */
       
      function LoginSubmit(value) {
        if( !input_Check() ){
          return false;
        }
        document.getElementById("IDToken1").value =
        document.getElementById("IDToken0").value +
        document.getElementById("IDToken1").value;
        aggSubmit();
        var hiddenFrm = document.forms['Login'];
        if (hiddenFrm != null) {
            hiddenFrm.elements['IDButton'].value = value;
            if (this.submitted) {
                alert("The request is currently being processed");
            } else {
                this.submitted = true;
                document.getElementById("submit_button").disabled = true;
		hiddenFrm.submit();
            }
        }
      }
  // Empty script so IE5.0 Windows will draw table and button borders
      function groupID_change(groupID){
        //グループ会社を選択したら、ウィンドウの会社コードを変更する
        document.getElementById("IDToken0").value = groupID; 
      }
      function input_Check(){
        var username = document.getElementById("IDToken1").value;
        var password = document.getElementById("IDToken2").value;
        var groupname = document.getElementById("IDToken0-0").value;
        var groupcode = document.getElementById("IDToken0").value;
        // groupID empty check (for Group)
        if ( "" == "group" ){
          if ( groupname.length == 0 || groupcode.length == 0 ){
            document.getElementById("b-message-empty").innerHTML = "グループ会社が選択されていません。";
            return false;
          }
        }
        // userid empty check
        if( username.length == 0 ){
          document.getElementById("b-message-empty").innerHTML = "ユーザＩＤが入力されていません。";
          return false;
        }
        // userid format check (for Obayashi)
         if ( "" == "obayashi" ){
          if ( username.length != 6 || !(username.match(/^U/gi )) ){
            if ( !(username.match(/^(A|B)/gi )) ){
              document.getElementById("b-message-empty").innerHTML = "ユーザＩＤの入力に誤りがあります。";
              return false;
            } else {
              if ( username.length < 5 || username.length > 15 ) {
                document.getElementById("b-message-empty").innerHTML = "ユーザＩＤの入力に誤りがあります。";
                return false;
              }
            }
          }
        } 
        // password empty check
        if( password.length == 0 ){
          document.getElementById("b-message-empty").innerHTML = "パスワードが入力されていません。";
          return false;
        }
        return true;
      }
      大林組イントラネットログイン画面
    ユーザIDとパスワードを半角で入力してください
    if (document.getElementById("IDToken1")) {
    if (document.getElementById("IDToken1").type == "password") {
      bgroupMiddle.setAttribute("class", "b-groupMiddle_only");
      bgroupMiddle.setAttribute("className", "b-groupMiddle_only");
    }
  }
  グループ会社
      大林組
      アトリエ・ジーアンドビー
      オーク情報システム
      オーク設備工業
      大林クリーンエナジー
      大林神栖バイオマス発電
      大月バイオマス発電
      大月ウッドサプライ
      大林新星和不動産
      大林デザインパートナーズ
      大林道路
      大林ファシリティーズ
      相馬環境サービス
      特研メカトロニクス
      内外テクノス
      ルポンドシエル
      ＰＬｉＢＯＴ
      大林リアルティマネジメント
      オプライゾン
      オークロード熊本
      富士防災
      サイプレス・スナダヤ
      オークロード高知
      オークロード広島
      ＭｉＴＡＳＵＮ
      築柴
      会社コード
    ユーザＩＤ
    if (document.getElementById("IDToken1")) {
    if (document.getElementById("IDToken1").type == "password") {
      bgroupMiddle.setAttribute("class", "b-groupMiddle_only");
      bgroupMiddle.setAttribute("className", "b-groupMiddle_only");
    }
  }
 パスワード
    if (document.getElementById("IDToken1")) {
    if (document.getElementById("IDToken1").type == "password") {
      bgroupMiddle.setAttribute("class", "b-groupMiddle_only");
      bgroupMiddle.setAttribute("className", "b-groupMiddle_only");
    }
  }
パスワードに入力した文字を確認したい場合はここをクリックしてください 
        ユーザIDがロックされたり、パスワードを忘れた場合
          （Windowsログオンのパスワードを忘れた場合もこちら）
        パスワードの変更方法
         パスワードに関する質問
      パスワードの変更は即時(数分程度)に反映されます。
      ※イントラネットに掲載されている情報には機密情報（社外秘、関係者外秘）が含まれますので、取り扱いには充分ご注意ください。
    パスワードに関する質問
        ユーザIDがロックされたり、パスワードを忘れた場合
      パスワードの変更は即時(数分程度)に反映されます。
      ※グループポータルに掲載されている情報には機密情報（社外秘、関係者外秘）が含まれますので、取り扱いには充分ご注意ください。
      Copyright(C) 2016 Obayashi Corporation. All Rights Reserved..

         if (elmCount != null) {
          document.write("
");
         }
                )

        )
    [isHtml:Symfony\Component\DomCrawler\Crawler:private] => 1
    [html5Parser:Symfony\Component\DomCrawler\Crawler:private] => 
)
