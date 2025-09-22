log
[2025-09-22 12:03:07] local.INFO: https://account.box.com/api/oauth2/authorize?response_type=code&client_id=&redirect_uri=  

in env
BOX_CLIENT_ID = f7044xrm37qwykr
BOX_CLIENT_SECRET = NMWYms9o
BOX_CALLBACK_URI = http://localhost:8080/ccc/box/callback


in controller

 $baseUrl = "https://account.box.com/api/oauth2/authorize?";
        $clientId = "";
        $secrectKey = "";
        $redirect_uri = "";
        if (strpos(url(''), 'deployment') !== false) {
            $clientId = env("BOX_CLIENT_ID_FOR_DEV_SLOT");
            $secrectKey = env("BOX_CLIENT_SECRET_FOR_DEV_SLOT");
            $redirect_uri = env("BOX_CALLBACK_URI_FOR_DEV_SLOT");
        } else {
            $clientId = env("BOX_CLIENT_ID");
            $secrectKey = env("BOX_CLIENT_SECRET");
            $redirect_uri = env("BOX_CALLBACK_URI");
        }

        session()->put("box_loggedin_page_url", $logged_in_url); //set current url to session
        session()->put("box_loggedin_search_states", $logged_in_search_states); //set search states to session ,its just working for 案件報告画面

        $url = $baseUrl . 'response_type=code' . '&client_id=' . $clientId . '&redirect_uri=' . $redirect_uri;
        Log::info($url);
        return array("url" => $url, "status" => "go to callback");
