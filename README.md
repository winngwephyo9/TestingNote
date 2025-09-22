
    /**
     * 【OBJ】DBキャッシュから、保存されているプロジェクトのリストを取得する
     */
    public function getCachedProjectList()
    {
        $projects = ModelFileCache::select('project_box_id', 'project_name')
            ->whereNotNull('project_name')->distinct()->get()
            ->map(function ($file) {
                return ['id' => $file->project_box_id, 'name' => $file->project_name];
            });
        return $projects->sortBy('name')->values();
    }

    /**
     * 【OBJ】DBキャッシュからモデルデータを取得する
     */
    public function getCachedModelData($projectFolderId)
    {
        $cachedFiles = ModelFileCache::where('project_box_id', $projectFolderId)->get()->groupBy('base_name');
        if ($cachedFiles->isEmpty()) {
            return ['error' => 'No cached model found. Please log in to Box to sync data.'];
        }
        return $this->formatDataForFrontend($cachedFiles);
    }


    /**
     * 【OBJ】同期時にプロジェクト名をDBに保存する
     */
    public function syncAndGetModelData($projectFolderId, $accessToken, $refreshToken, $urlPath, $boxLoginTime)
    {
        mb_internal_encoding('UTF-8');
        set_time_limit(0);
        $this->projectFolderId = $projectFolderId;
        $this->accessToken = $accessToken;
        $this->refreshToken = $refreshToken;
        $this->urlPath = $urlPath;
        $this->boxLoginTime = $boxLoginTime;
        try {
            $this->wnpcheckBoxExpiryStatus();
            // $accessToken = session('access_token');
            // if (empty($accessToken)) throw new Exception("Box access token is missing.");

            $client = new Client(['verify' => false]);
            $projectName = $this->fetchProjectName($this->projectFolderId, $this->accessToken, $client);

            $boxFiles = $this->fetchFullBoxFileList($this->projectFolderId, $this->accessToken, $client);
            $boxFilesById = collect($boxFiles)->keyBy('id');
            $dbFiles = ModelFileCache::where('project_box_id', $projectFolderId)->get()->keyBy('box_file_id');

            $filesToUpdate = [];
            foreach ($boxFiles as $boxFile) {
                $dbFile = $dbFiles->get($boxFile['id']);
                $boxModifiedAtJst = Carbon::parse($boxFile['modified_at'])->setTimezone('Asia/Tokyo')->startOfSecond();
                $dbModifiedAtJst = $dbFile ? $dbFile->box_modified_at->startOfSecond() : null;

                if (!$dbFile || $dbModifiedAtJst->lt($boxModifiedAtJst)) {
                    $filesToUpdate[] = $boxFile;
                }
            }

            if (empty($filesToUpdate)) {
                Log::info("No files need to be updated. Sync is complete.");
                return 0;
            }

            Log::info("Found " . count($filesToUpdate) . " files to update/insert.");
            $updateChunks = array_chunk($filesToUpdate, 500);

            foreach ($updateChunks as $chunkIndex => $chunkOfFiles) {
                Log::info("Processing chunk " . ($chunkIndex + 1) . "/" . count($updateChunks) . " (" . count($chunkOfFiles) . " files)...");

                // ダウンロードとリフレッシュを自動で処理するメソッドを呼び出す
                $downloadedContents = $this->fetchContentsWithRefresh($chunkOfFiles);

                $dataToUpsert = [];
                foreach ($downloadedContents as $fileId => $content) {
                    $boxFile = $boxFilesById->get($fileId);
                    if ($boxFile) {
                        $dataToUpsert[] = [
                            'project_box_id' => $projectFolderId,
                            'project_name' => $projectName,
                            'file_name' => $boxFile['name'],
                            'base_name' => pathinfo($boxFile['name'], PATHINFO_FILENAME),
                            'box_file_id' => $fileId,
                            'file_type' => strtolower(pathinfo($boxFile['name'], PATHINFO_EXTENSION)),
                            'content' => $content,
                            'box_modified_at' => Carbon::parse($boxFile['modified_at'])->setTimezone('Asia/Tokyo'),
                            'created_at' => now(),
                            'updated_at' => now(),
                        ];
                    }
                }
                if (!empty($dataToUpsert)) {
                    ModelFileCache::upsert($dataToUpsert, ['box_file_id'], ['project_name', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at']);
                }
            }

            $this->cleanupDeletedFiles($projectFolderId, $boxFilesById);

            return count($filesToUpdate);
        } catch (Throwable $e) {
            Log::error("Box Sync Failed: " . $e->getMessage(), ['exception' => $e]);
            throw $e;
        }
    }

    /**
     * 【OBJ】削除されたファイルをDBとストレージからクリーンアップする
     */
    private function cleanupDeletedFiles($projectFolderId, $boxFilesById)
    {
        $dbFileIds = ModelFileCache::where('project_box_id', $projectFolderId)->pluck('box_file_id');
        $boxFileIds = $boxFilesById->keys();
        $deletedIds = $dbFileIds->diff($boxFileIds)->all();

        if (!empty($deletedIds)) {
            Log::info("Deleting " . count($deletedIds) . " records from database...");
            ModelFileCache::whereIn('box_file_id', $deletedIds)->delete();
        }
    }

    /**
     * 【OBJ】Boxからプロジェクト名（フォルダ名）を取得する
     */
    private function fetchProjectName($projectFolderId, $accessToken, Client $client)
    {
        $header = ["Authorization" => "Bearer " . $accessToken];
        $folderInfoUrl = "https://api.box.com/2.0/folders/{$projectFolderId}?fields=name";
        $folderInfoResponse = $client->get($folderInfoUrl, ['headers' => $header]);
        return json_decode($folderInfoResponse->getBody()->getContents())->name;
    }

    /**
     * 【OBJ】フロントエンド用にデータを整形する
     */
    private function formatDataForFrontend($groupedFiles)
    {
        $objMtlPairs = [];
        foreach ($groupedFiles as $baseName => $files) {
            $objFile = $files->firstWhere('file_type', 'obj');
            $mtlFile = $files->firstWhere('file_type', 'mtl');
            if ($objFile) {
                $objMtlPairs[] = [
                    'baseName' => $baseName,
                    'obj' => ['name' => $objFile->file_name, 'content' => $objFile->content],
                    'mtl' => $mtlFile ? ['name' => $mtlFile->file_name, 'content' => $mtlFile->content] : null
                ];
            }
        }
        return $objMtlPairs;
    }

    /**
     * 【OBJ】Boxからファイルリストを全て取得（更新日時も含む）
     */
    private function fetchFullBoxFileList($projectFolderId, $accessToken, $client)
    {
        $header = ["Authorization" => "Bearer " . $accessToken];
        $allFolderItems = [];
        $offset = 0;
        $limit = 1000;
        do {
            $requestURL = "https://api.box.com/2.0/folders/{$projectFolderId}/items?fields=id,name,modified_at&limit={$limit}&offset={$offset}";
            $response = $client->get($requestURL, ['headers' => $header]);
            $body = json_decode($response->getBody()->getContents(), true);
            if (isset($body['entries'])) {
                $allFolderItems = array_merge($allFolderItems, $body['entries']);
                $offset += count($body['entries']);
            } else {
                break;
            }
        } while (isset($body['total_count']) && $offset < $body['total_count']);
        return $allFolderItems;
    }

    /**
     * 【OBJ】ダウンロードとトークンリフレッシュを自動で処理するラッパー
     */
    private function fetchContentsWithRefresh(array $filesToDownload)
    {
        $maxRetries = 3;
        for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
            $result = $this->fetchMultipleBoxFileContentsConcurrently($filesToDownload, $this->accessToken);

            if ($result['status'] === 'success') {
                return $result['contents']; // 成功したらすぐに返す
            }

            if ($result['status'] === 'token_expired') {
                $this->wnpcheckBoxExpiryStatus();
                // Log::info("Token expired. Attempting refresh (Attempt {$attempt}/{$maxRetries})");
                // $newTokens = $this->refreshBoxToken($refreshToken, $urlPath);
                // if (!$newTokens) {
                //     throw new Exception("Failed to refresh Box token after expiration.");
                // }
                // // セッションを更新して、次のループで再試行
                // //  **【重要】ローカル変数を新しいトークンで上書きする**
                // $accessToken = $newTokens['access_token'];
                // $refreshToken = $newTokens['refresh_token'];
                continue;
            }

            // その他のエラー
            throw new Exception($result['message'] ?? 'Unknown download error.');
        }

        throw new Exception("Failed to download files after {$maxRetries} refresh attempts.");
    }

    /**
     * 【OBJ】Guzzle Poolに渡すリクエストの形式を修正
     */
    private function fetchMultipleBoxFileContentsConcurrently(array $files, $accessToken)
    {
        $downloadedContents = [];
        $tokenExpired = false;

        $client = new Client(['verify' => false]);
        $header = ["Authorization" => "Bearer " . $accessToken];

        //  **【重要】リクエストジェネレータの修正**
        $requests = function ($files) use ($header) {
            foreach ($files as $file) {
                $url = "https://api.box.com/2.0/files/{$file['id']}/content";
                // GuzzleRequestオブジェクトそのものをyieldする
                yield $file['id'] => new GuzzleRequest('GET', $url, $header);
            }
        };

        $pool = new Pool($client, $requests($files), [
            'concurrency' => 10,
            'fulfilled' => function ($response, $fileId) use (&$downloadedContents) {
                // fulfilledはキー（fileId）を受け取るので、マッピングは不要
                $downloadedContents[$fileId] = $response->getBody()->getContents();
            },
            'rejected' => function ($reason, $fileId) use (&$tokenExpired) {
                if ($reason instanceof ClientException && $reason->getResponse()->getStatusCode() == 401) {
                    $tokenExpired = true;
                } else {
                    Log::error("Failed to download file ID {$fileId}: " . $reason->getMessage());
                }
            }
        ]);

        $promise = $pool->promise();
        $promise->wait();

        if ($tokenExpired) {
            return ['status' => 'token_expired', 'contents' => []];
        }

        return ['status' => 'success', 'contents' => $downloadedContents];
    }


    /**
     * 【新規】DBから最新のBoxトークン情報を取得する
     */
    public function getLatestBoxTokens()
    {
        return DB::connection('dldwh')->table('box_tokens')->first();
    }



    public function refreshBoxToken($refreshToken, $urlPath)
    {
        if (empty($refreshToken)) {
            Log::error("Refresh token is empty.");
            return null;
        }
        try {
            $clientId = env("BOX_CLIENT_ID");
            $clientSecret = env("BOX_CLIENT_SECRET");
            if (strpos($urlPath, 'deployment') !== false) {
                $clientId = env("BOX_CLIENT_ID_FOR_DEV_SLOT");
                $clientSecret = env("BOX_CLIENT_SECRET_FOR_DEV_SLOT");
            }
            if (!$clientId || !$clientSecret) {
                Log::error("Box client credentials are not set.");
                return null;
            }
            $response = Http::asForm()->post('https://api.box.com/oauth2/token', [
                'grant_type' => 'refresh_token',
                'refresh_token' => $refreshToken,
                'client_id' => $clientId,
                'client_secret' => $clientSecret,
            ]);
            if ($response->successful()) {
                $data = $response->json();
                $newAccessToken = $data['access_token'];
                $newRefreshToken = $data['refresh_token'];

                // 【重要】DBとセッションの両方を更新
                $this->saveBoxTokens($newAccessToken, $newRefreshToken);
                session(['access_token' => $newAccessToken, 'refresh_token' => $newRefreshToken, 'box_login_time' => time()]);

                Log::info("Successfully refreshed and saved Box tokens.");
                return ['access_token' => $newAccessToken, 'refresh_token' => $newRefreshToken];
            } else {
                Log::error("Failed to refresh Box token.", ['status' => $response->status(), 'body' => $response->body()]);
                return null;
            }
        } catch (Exception $e) {
            Log::error("Exception on refresh token: " . $e->getMessage());
            return null;
        }
    }

    /**
     * saveBoxExpiryStatus
     * トーケンを更新した後、記録する
     *
     * @return void
     */
    // public function saveBoxExpiryStatus()
    // {
    //     $query = "INSERT INTO box_expiry_status (status) VALUES (?)";
    //     $result = DB::connection('dldwh')->insert($query, ['true']);
    // }

    /**
     * ジョブ内でトークンの有効期限をチェックし、更新するメソッド
     */
    // public function checkBoxExpiryStatus()
    // {
    //     $loginTime = session('box_login_time', 0);
    //     if (time() > $loginTime + (60 * 50)) {
    //         Log::info('Box token is about to expire. Refreshing...');
    //         // refreshBoxTokenは中でDBとセッションを両方更新してくれる
    //         $newTokens = $this->refreshBoxToken(session('refresh_token'), session('url_path'));
    //         if (!$newTokens) {
    //             throw new Exception("Failed to refresh Box token.");
    //         }
    //     }
    // }

    /**
     * プロジェクトIDをキーとして、完了または失敗したジョブのログを取得する
     */
    public function getFinishedJobLog($projectId)
    {
        return DB::connection('dldwh')->table('job_logs_bara')->where('job_identifier', $projectId)->first();
    }

    /**
     * プロジェクトIDを指定して、古いジョブログを削除する
     */
    public function deleteFinishedJobLog($projectId)
    {
        DB::connection('dldwh')->table('job_logs_bara')->where('job_identifier', $projectId)->delete();
    }

    /**
     * Summary of jobLogsUpdateOrInsert
     * @param mixed $projectFolderId
     * @param mixed $status
     * @param mixed $message
     * @return void
     */
    public function jobLogsUpdateOrInsert($projectFolderId, $status, $message)
    {
        DB::connection('dldwh')->table('job_logs_bara')->updateOrInsert(
            ['job_identifier' => $projectFolderId],
            ['status' => $status, 'message' => $message, 'updated_at' => now()]
        );
    }

    /**
     * 【OBJ】
     * 
     * @return void
     */
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
            Log::info('New accessToken : ' . $this->accessToken);
            Log::info('New refreshToken : ' . $this->refreshToken);
            $this->wnpsaveAccessToken();


            $this->saveBoxExpiryStatus();
            $this->boxLoginTime = time();
        }
    }

    /**
     * 【OBJ】saveAccessToken
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
        Log::info('Session New accessToken : ' . session('access_token'));
        Log::info('Session New refreshToken : ' . session('refresh_token'));
        $this->saveBoxTokens();
        // session(['access_token' => $newAccessToken, 'refresh_token' => $newRefreshToken, 'box_login_time' => time()]);

    }

    /**
     * 【新規】DBに新しいBoxトークン情報を保存/更新する
     */
    public function saveBoxTokens()
    {
        DB::connection('dldwh')->table('box_tokens')->updateOrInsert(
            ['id' => 1], // 常にID=1の行を更新
            [
                'access_token' => $this->accessToken,
                'refresh_token' => $this->refreshToken,
                'login_time' => time(),
                'updated_at' => now()
            ]
        );
    }

    /**
     * 【OBJ】saveBoxExpiryStatus
     * トーケンを更新した後、記録する
     *
     * @return void
     */
    public function saveBoxExpiryStatus()
    {
        $query = "INSERT INTO box_expiry_status (status) VALUES (?)";
        $result = DB::connection('dldwh')->insert($query, ['true']);
    }
