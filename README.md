
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
            $this->checkBoxExpiryStatus();
            $client = new Client(['verify' => false]);
            $projectName = $this->fetchProjectName($client);
            $boxFiles = $this->fetchFullBoxFileList($client);
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
     * 【OBJ】Boxからプロジェクト名（フォルダ名）を取得する
     */
    private function fetchProjectName(Client $client)
    {
        $header = ["Authorization" => "Bearer " . $this->accessToken];
        $folderInfoUrl = "https://api.box.com/2.0/folders/{$this->projectFolderId}?fields=name";
        $folderInfoResponse = $client->get($folderInfoUrl, ['headers' => $header]);
        return json_decode($folderInfoResponse->getBody()->getContents())->name;
    }

    /**
     * 【OBJ】Boxからファイルリストを全て取得（更新日時も含む）
     */
    private function fetchFullBoxFileList($client)
    {
        $header = ["Authorization" => "Bearer " . $this->accessToken];
        $allFolderItems = [];
        $offset = 0;
        $limit = 1000;
        do {
            $requestURL = "https://api.box.com/2.0/folders/{$this->projectFolderId}/items?fields=id,name,modified_at&limit={$limit}&offset={$offset}";
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
                $this->checkBoxExpiryStatus();
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
