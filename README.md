[2025-09-22 12:28:03] local.INFO: Token is about to expire  
[2025-09-22 12:28:03] local.ERROR: Box Sync Failed: Call to a member function post() on null {"exception":"[object] (Error(code: 0): Call to a member function post() on null at C:\\xampp\\htdocs\\CCC\\ccc\\app\\Models\\DLDHWDataImportModel.php:5752)
[stacktrace]
#0 C:\\xampp\\htdocs\\CCC\\ccc\\app\\Models\\DLDHWDataImportModel.php(5390): App\\Models\\DLDHWDataImportModel->wnpcheckBoxExpiryStatus()
#1 C:\\xampp\\htdocs\\CCC\\ccc\\app\\Jobs\\SyncBoxProject.php(80): App\\Models\\DLDHWDataImportModel->syncAndGetModelData('341288887011', 'JfLmqszl9Rv2TcW...', 'IYF5XN6U6wTdvvm...', 'http://localhos...', NULL)
#2 C:\\xampp\\htdocs\\CCC\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Container\\BoundMethod.php(36): App\\Jobs\\SyncBoxProject->handle()


public function syncAndGetModelData($projectFolderId, $accessToken, $refreshToken, $urlPath, $boxLoginTime)
    {
        mb_internal_encoding('UTF-8');
        set_time_limit(0);
        Log::info("Url MOdel >>>>>" . $urlPath);
        $this->projectFolderId = $projectFolderId;
        $this->accessToken = $accessToken;
        $this->refreshToken = $refreshToken;
        $this->urlPath = $urlPath;
        $this->boxLoginTime = $boxLoginTime;
        try {
            $this->wnpcheckBoxExpiryStatus();
  and another code
  }
public function wnpcheckBoxExpiryStatus()
    {
        $tokenExpiryTime = $this->boxLoginTime + 3600; //box期限60分
        $clientId = "";
        $secrectKey = "";
        $accessToken = "";

        if (strpos($this->url_path, 'deployment') !== false) {
            $clientId = env("BOX_CLIENT_ID_FOR_DEV_SLOT");
            $secrectKey = env("BOX_CLIENT_SECRET_FOR_DEV_SLOT");
        } else {
            $clientId = env("BOX_CLIENT_ID");
            $secrectKey = env("BOX_CLIENT_SECRET");
        }

        // トークンの有効期限が10分未満の場合、新しいトークンを取得
        if (time() > $tokenExpiryTime - 600) { // 10分未満の場合
            Log::info('Token is about to expire');
            $response = $this->client->post('https://api.box.com/oauth2/token', [
                'form_params' => [
                    'grant_type' => 'refresh_token',
                    'client_id' => $clientId,
                    'client_secret' => $secrectKey,
                    'refresh_token' => $this->refresh_token,
                ],
            ]);

            $data = json_decode($response->getBody(), true);
            $accessToken = $data['access_token'];
            $refreshToken = $data['refresh_token'];
            $this->refresh_token = $refreshToken;
            $this->access_token = $accessToken;

            $this->wnpsaveAccessToken($accessToken, $refreshToken);
            Log::info('accessToken : ' . $accessToken);
            Log::info('refreshToken : ' . $refreshToken);

            $this->saveBoxExpiryStatus();
            $this->boxLoginTime = time();
        }
    }
