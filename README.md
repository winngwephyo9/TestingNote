     $client = new \GuzzleHttp\Client([
            'allow_redirects' => true,
            'cookies' => true, // Maintain session
        ]);

        $baseUrl = 'http://www.fc.obayashi.co.jp/'; // Replace with the base URL

        try {
            // Step 1: Login
            $loginUrl = 'https://cqms.obayashi.co.jp/sso/UI/Login'; // Replace with actual action URL
            $loginResponse = $client->post($loginUrl, [
                'headers' => [
                    'User-Agent' => 'Mozilla/5.0',
                    'Accept' => 'text/html,application/xhtml+xml',
                    'Referer' => 'https://cqms.obayashi.co.jp/', // optional but helpful
                ],
                'form_params' => [
                    'IDToken1' => 'UHR757',
                    'IDToken2' => 'Win22222',
                    'goto' => '...', // include hidden fields if required
                    'type' => 'obayashi',
                    'encoded' => 'true',
                    'gx_charset' => 'UTF-8',
                ],
            ]);
            // log::info("Login Response >>>>>>>>>>>>>>>>>>>>");
            // Safely encode to JSON
            log::info(json_encode($loginResponse));

            // Check for successful login (you might need to inspect the response)
            $html = (string) $loginResponse->getBody();

            log::info("Full HTML: " . $html);

            // log::info("Login HTML: " . substr($html, 0, 500)); // Log first 500 chars

            if (strpos($html, 'successful_login_indicator') !== false) {
                log::info("Login successful!");
            } else {
                log::error("Login failed or indicator not found.");
            }

  
<input name="__RequestVerificationToken" type="hidden" value="pIpn-l9VYaB-GLzYK7q9C8dtNKbqD1eAtO4JgpxlLJjPzbBGwlMfLTXXhn-2zMRXEXfBE3RppFehBX58lTfuOl_frclIOeiELh-NqvlByg81" />

<html>
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>ログイン</title>

    <!-- include css -->
    <link href="/Contents/css/bootstrap.min.css" rel="stylesheet" />
    <link href="/Contents/css/all.min.css" rel="stylesheet" />
    <link href="/Contents/css/style.css" rel="stylesheet" />
    <!-- END: include css -->
</head>
<body>
    <header>
        <h1 class="title">受入承認システム</h1>
        
        
    </header><!-- END: header -->

    <div class="main-content">
        <div class="form-user login" style="left: 50%;top: 35%; ">
            <form>
                <div class="form-group">
                    <label>メールアドレス</label>
                    <input type="text" class="form-control" name="ID" id="txtUsername" maxlength="100">
                </div>
                <div class="form-group">
                    <label>パスワード</label>
                    <input type="password" class="form-control" name="password" id="txtPassword" maxlength="32">
                </div>
                <div class="form-group text-right button">
                    <button type="button" id="btnLogin" class="btn green-btn" style="width:100%">ログイン</button>
                </div>
            </form>
            <div class="forgot"><a href="/HomeCQMS/ForgetPass">大林組職員以外の方でパスワードをお忘れの場合</a></div>

            <div class="forgot register"><a href="/HomeCQMS/Register">新規ユーザー登録</a></div>
            <br/>
            <div class="form-group">
                <textarea class="form-control" id="txtareaMessage" rows="6" cols="100" readonly>
サーバーメンテナンス実施日：
2019年10月23日 0:00～1:00
3.
4.
5.
6.
                </textarea>
            </div>
        </div>
    </div><!-- END: .main-content -->
    <div id="loadding" class="loading-no-action">
        <div id="loader"></div>
    </div>
    <!-- The Modal Success -->
    <div class="modal" id="md-success">
        <div class="modal-dialog">
            <div class="modal-content">

                <!-- Modal Header -->
                <div class="modal-header">
                    <h4 class="modal-title">
                        <i class="fa fa-check-circle"></i>
                        <span>成功</span>
                    </h4>
                    
                </div>

                <!-- Modal body -->
                <div class="modal-body" id="md-msgSs">
                </div>

                <!-- Modal footer -->
                <div class="modal-footer">
                    <button type="button" class="btn btn-sm" style="width: 130px;" data-dismiss="modal" id="ss-btnOK">OK</button>
                </div>
            </div>
        </div>
    </div>
    <!-- The Modal Error -->
    <div class="modal" id="md-error">
        <div class="modal-dialog">
            <div class="modal-content">

                <!-- Modal Header -->
                <div class="modal-header">
                    <h4 class="modal-title">
                        <i class="fa fa-exclamation-circle"></i>
                        <span>エラー</span>
                    </h4>
                    
                </div>

                <!-- Modal body -->
                <div class="modal-body" id="md-msgEr">
                </div>

                <!-- Modal footer -->
                <div class="modal-footer">
                    <button type="button" class="btn btn-sm" style="width: 130px;" data-dismiss="modal" id="er-btnOK">OK</button>
                </div>
            </div>
        </div>
    </div>
    <footer>
        <div class="footer-left">
            Version: Web:1.4.1 [iOS:1.4.0対応]
        </div>
        <div class="footer-right">
            Copyright (c) OBAYASHI CORPORATION.
        </div>
    </footer><!-- END: footer -->
    <!-- include javascript -->
    <script src="/Scripts/jquery-2.2.4.min.js"></script>
    <script src="/Scripts/jquery-url-min.js"></script>
    <script src="/Contents/js/bootstrap.min.js"></script>

<script src="/bundles/Contents/main?v=SbSAR7Mj03np5zDJe8Ixb_2uYXkLwZTNmH7c2RdlcS41"></script>
<script src="/bundles/controllers/StringFormat?v=Y3hFCd5pb9PzTP8Z179NqcSmvNicT3Vd2w-kiZQVIaE1"></script>
<script src="/bundles/controllers/MessageRes?v=a1WoG2jHLCVg_P3cDcfI7B307I764Zc6U69VmIkcNiM1"></script>

<script src="/bundles/controllers/ADFSAuthorizeController?v=mhiaHP4sNCetyPBaSxZ6kgHFysxlenAXrCqBpVwBLi01"></script>

    <script>
        (function () {
            AuthorizeController.init();
            //
            GetMessage();
        })();
        function GetMessage() {
            $("#loadding").show();
            $.ajax({
                type: "POST",
                url: "/SystemTop/GetMessage",
                contentType: "application/json; charset=utf-8",
                dataType: "json",
                headers: {
                    'token': $("input[name=__RequestVerificationToken]").val()
                },
                success: function (response) {
                    $("#loadding").hide();
                    if (response.Success) {
                        $('#txtareaMessage').html(function () { return response.Message; });
                    } else {
                        if (response.IsSessionTimeOut) {
                            var msg1 = localStorage.getItem('Msg082');
                            $("#md-error").modal({ backdrop: 'static', keyboard: false });
                            $("#md-msgEr").html(msg1);
                            $('#er-url').val("/");
                        } else {
                            var msg2 = localStorage.getItem('Msg047');
                            $("#md-error").modal({ backdrop: 'static', keyboard: false });
                            $("#md-msgEr").html(msg2);
                            $('#er-url').val("/");
                        }
                    }
                },
                error: function (err) {
                    $("#loadding").hide();
                    var msg = localStorage.getItem('Msg081');
                    $("#md-error").modal({ backdrop: 'static', keyboard: false });
                    $("#md-msgEr").html(msg);
                    $('#er-url').val("/");
                    $(this).removeAttr("disabled");
                }
            });
        }
    </script>
    <!-- END: include javascript -->
</body>
</html>  
[2025-05-13 10:41:16] local.ERROR: Login failed or indicator not found.  

