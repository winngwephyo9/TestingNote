 public function refreshBoxToken($refreshToken, $urlPath)
    {
        if (empty($refreshToken)) {
            Log::error("Refresh token is empty. Cannot refresh.");
            return null;
        }

        try {
            // .envのキーを、本番環境かステージング環境かで切り替える
            $clientId = env("BOX_CLIENT_ID");
            $clientSecret = env("BOX_CLIENT_SECRET");
            
            // 'deployment'という文字列がURLに含まれていれば、ステージング用のキーを使用
            if (strpos($urlPath, 'deployment') !== false) {
                $clientId = env("BOX_CLIENT_ID_FOR_DEV_SLOT");
                $clientSecret = env("BOX_CLIENT_SECRET_FOR_DEV_SLOT");
            }

            if (!$clientId || !$clientSecret) {
                Log::error("Box client credentials are not set for the current environment.");
                return null;
            }

            // LaravelのHTTPクライアントを使ってBox APIにリクエストを送信
            $response = Http::asForm()->post('https://api.box.com/oauth2/token', [
                'grant_type' => 'refresh_token',
                'refresh_token' => $refreshToken,
                'client_id' => $clientId,
                'client_secret' => $clientSecret,
            ]);

            if ($response->successful()) {
                // リフレッシュ成功
                $data = $response->json();
                $newAccessToken = $data['access_token'];
                $newRefreshToken = $data['refresh_token'];

                // 1. 新しいトークンをデータベースに保存（一元管理）
                $this->saveBoxTokens($newAccessToken, $newRefreshToken);
                
                // 2. ユーザーのブラウザセッションも更新（次回のUI操作のため）
                session([
                    'access_token' => $newAccessToken,
                    'refresh_token' => $newRefreshToken,
                    'box_login_time' => time(),
                ]);

                Log::info("Successfully refreshed and saved new Box tokens to the database and session.");
                
                // 3. 新しいトークンのペアを呼び出し元に返す
                return [
                    'access_token' => $newAccessToken,
                    'refresh_token' => $newRefreshToken,
                ];

            } else {
                // リフレッシュ失敗
                $body = $response->body();
                Log::error("Failed to refresh Box token.", [
                    'status' => $response->status(),
                    'body' => $body
                ]);

                // もしリフレッシュトークン自体が無効なら、DBから削除する
                if (strpos($body, 'invalid_grant') !== false) {
                    $this->deleteBoxTokens();
                }

                return null;
            }
        } catch (Exception $e) {
            Log::error("An exception occurred while refreshing Box token: " . $e->getMessage());
            return null;
        }
    }
