In the controller I write like this
            if (session()->has('refresh_token')) {
                $this->refresh_token = session('refresh_token');
                Log::info('refresh_token ' . $this->refresh_token);
            }
In the model i write like this
  public function wnpcheckBoxExpiryStatus()
    {
        $tokenExpiryTime = $this->boxLoginTime + 3600; //box期限60分
        $clientId = "";
        $secrectKey = "";
        $accessToken = "";
        $client = new Client(['verify' => false]);

        if (strpos($this->urlPath, 'deployment') !== false) {
            $clientId = env("BOX_CLIENT_ID_FOR_DEV_SLOT");
            $secrectKey = env("BOX_CLIENT_SECRET_FOR_DEV_SLOT");
        } else {
            $clientId = env("BOX_CLIENT_ID");
            $secrectKey = env("BOX_CLIENT_SECRET");
        }

        // トークンの有効期限が10分未満の場合、新しいトークンを取得
        if (time() > $tokenExpiryTime - 600) { // 10分未満の場合
            Log::info('Token is about to expire');
            $response = $client->post('https://api.box.com/oauth2/token', [
                'form_params' => [
                    'grant_type' => 'refresh_token',
                    'client_id' => $clientId,
                    'client_secret' => $secrectKey,
                    'refresh_token' => $this->refreshToken,
                ],
            ]);

            $data = json_decode($response->getBody(), true);
            $accessToken = $data['access_token'];
            $refreshToken = $data['refresh_token'];
            $this->refreshToken = $refreshToken;
            $this->accessToken = $accessToken;

            $this->wnpsaveAccessToken();
            Log::info('accessToken : ' . $this->accessToken);
            Log::info('refreshToken : ' . $this->refreshToken);

            $this->saveBoxExpiryStatus();
            $this->boxLoginTime = time();
        }
    }

/**
     * saveAccessToken
     * 
     * BOXログイン有効時間切れた後、更新したアクセストーケンを再保存する
     *
     * @param string $accessToken
     * @param string $refreshToken
     * @return void
     */
    public function wnpsaveAccessToken()
    {
        session()->forget('access_token');
        session()->forget('refresh_token');
        session(['access_token' => $this->accessToken]);
        session(['refresh_token' => $this->refreshToken]);
    }

    /**
     * saveBoxExpiryStatus
     * トーケンを更新した後、記録する
     *
     * @return void
     */
    public function saveBoxExpiryStatus()
    {
        $query = "INSERT INTO box_expiry_status (status) VALUES (?)";
        $result = DB::connection('dldwh')->insert($query, ['true']);
    }

In the log
13:00:37] local.INFO: refresh_token jL3FAIx7VCmu7cMHTKyohLnsvlsH0kfD0kRZqBepPXA8tqosS5SWkQGzpJ0pKAMA  
[2025-09-22 13:00:37] local.INFO: url_path http://localhost:8080/ccc  
[2025-09-22 13:00:37] local.INFO: Url Controller >>>>>http://localhost:8080/ccc  
[2025-09-22 13:00:38] local.INFO: Job started for project: 341288887011  
[2025-09-22 13:00:38] local.INFO: Url MOdel >>>>>http://localhost:8080/ccc  
[2025-09-22 13:00:38] local.INFO: Token is about to expire  
[2025-09-22 13:00:38] local.INFO: accessToken : VuZ2rVG6QBS4EB72pSFUzq1yzue8YrAV  
[2025-09-22 13:00:38] local.INFO: refreshToken : 8pxCexjrNK0WuUo6PoVRz9UrymrYtuogVBRt6u3N2BIO38LdOxnFSosJBlYav7cz  
[2025-09-22 13:02:21] local.INFO: No files need to be updated. Sync is complete.  
[2025-09-22 13:02:21] local.INFO: Job completed for project 341288887011 with status: no_files_to_update  
[2025-09-22 13:02:43] local.INFO: box_login_time 1758513629  
[2025-09-22 13:02:43] local.INFO: refresh_token jL3FAIx7VCmu7cMHTKyohLnsvlsH0kfD0kRZqBepPXA8tqosS5SWkQGzpJ0pKAMA  
[2025-09-22 13:02:43] local.INFO: url_path http://localhost:8080/ccc  
[2025-09-22 13:02:43] local.INFO: Url Controller >>>>>http://localhost:8080/ccc  
[2025-09-22 13:03:17] production.ERROR: No application encryption key has been specified. {"exception":"[object] (Illuminate\\Encryption\\MissingAppKeyException(code: 0): No application encryption key has been specified. at C:\\xampp\\htdocs\\CCC\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Encryption\\EncryptionServiceProvider.php:101)
