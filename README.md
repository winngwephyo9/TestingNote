In the other category generate refresh token like below this checkBoxExpiryStatus function call before data insert into the table so 
base on the below code refactor current code
In the job file
    $this->checkBoxExpiryStatus();

        /**
     * checkBoxExpiryStatus
     * BOXログイン有効時間を確認し、切れる恐れがある場合、トーケンを更新する
     *
     * @return void
     */
    public function checkBoxExpiryStatus()
    {
        $dldwhImportController = new DLDWHDataImportController();
        $tokenExpiryTime = $this->box_login_time + 3600; //box期限60分
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

            $dldwhImportController->saveAccessToken($accessToken, $refreshToken);
            Log::info('accessToken : ' . $accessToken);
            Log::info('refreshToken : ' . $refreshToken);

            $this->saveBoxExpiryStatus();
            $this->box_login_time = time();
        }
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

    In the controller
    /**
     * saveAccessToken
     * 
     * BOXログイン有効時間切れた後、更新したアクセストーケンを再保存する
     *
     * @param string $accessToken
     * @param string $refreshToken
     * @return void
     */
    public function saveAccessToken($accessToken, $refreshToken)
    {
        session()->forget('access_token');
        session()->forget('refresh_token');
        session(['access_token' => $accessToken]);
        session(['refresh_token' => $refreshToken]);
    }


