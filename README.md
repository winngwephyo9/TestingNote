  /**
     * 【新規】Boxにログインしていない時に、DBキャッシュからモデルデータを取得する
     */
    public function getCachedModelData($projectFolderId)
    {
        // ModelFileCacheが自動的に'dldwh'接続を使用する
        $cachedFiles = ModelFileCache::where('project_box_id', $projectFolderId)
            ->get()
            ->groupBy('base_name');

        if ($cachedFiles->isEmpty()) {
            return ['error' => 'No cached model found. Please log in to Box to sync data for the first time.'];
        }

        return $this->formatDataForFrontend($cachedFiles);
    }

    /**
     * **【修正版】並列ダウンロードでタイムアウトを解決**
     * Boxにログインしている時に、BoxとDBを同期し、最新のモデルデータを取得する
     */
    public function syncAndGetModelData($projectFolderId)
    {
        // PHPの内部エンコーディングをUTF-8に設定（文字化け対策）
        mb_internal_encoding('UTF-8');
        // 多数のファイルをダウンロードするため、実行時間制限を無効化
        set_time_limit(0);

        Log::info("Starting sync process for project: {$projectFolderId}");

        try {
            // 1. Boxから最新のファイルリストを取得
            $boxFiles = $this->fetchFullBoxFileList($projectFolderId);
            $boxFilesById = collect($boxFiles)->keyBy('id');

            // 2. DBから現在のファイルリストを取得
            $dbFiles = ModelFileCache::where('project_box_id', $projectFolderId)
                ->get()->keyBy('box_file_id');

            // 3. 更新または追加が必要なファイルのリストを作成
            $filesToUpdate = [];
            foreach ($boxFiles as $boxFile) {
                $dbFile = $dbFiles->get($boxFile->id);
                if (!$dbFile || new \DateTime($dbFile->box_modified_at) < new \DateTime($boxFile->modified_at)) {
                    $filesToUpdate[] = $boxFile;
                }
            }

            // 4. 更新が必要なファイルがあれば、並列でコンテンツをダウンロード
            if (!empty($filesToUpdate)) {
                Log::info("Found " . count($filesToUpdate) . " files to update/insert. Starting download...");
                $downloadedContents = $this->fetchMultipleBoxFileContentsConcurrently($filesToUpdate);
                Log::info("Finished downloading " . count($downloadedContents) . " files.");

                // 5. UPSERT用のデータ配列を作成
                $dataToUpsert = [];
                foreach ($downloadedContents as $fileId => $content) {
                    $boxFile = $boxFilesById->get($fileId);
                    if ($boxFile) {
                        $dataToUpsert[] = [
                            'project_box_id' => $projectFolderId,
                            'file_name' => $boxFile->name,
                            'base_name' => pathinfo($boxFile->name, PATHINFO_FILENAME),
                            'box_file_id' => $fileId,
                            'file_type' => strtolower(pathinfo($boxFile->name, PATHINFO_EXTENSION)),
                            'content' => $content,
                            'box_modified_at' => new \DateTime($boxFile->modified_at),
                            'created_at' => now(),
                            'updated_at' => now(),
                        ];
                    }
                }

                // 6. データを500件ずつのチャンクに分割してUpsertする（プレースホルダ上限エラー回避）
                if (!empty($dataToUpsert)) {
                    $chunks = array_chunk($dataToUpsert, 500);

                    Log::info("Upserting " . count($dataToUpsert) . " records in " . count($chunks) . " chunks...");
                    foreach ($chunks as $index => $chunk) {
                        ModelFileCache::upsert(
                            $chunk,
                            ['box_file_id'], // 一意キー
                            ['project_box_id', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at'] // 更新対象カラム
                        );
                        Log::info("Upserted chunk " . ($index + 1) . "/" . count($chunks) . " (" . count($chunk) . " records).");
                    }
                    Log::info("Upsert complete.");
                }
            } else {
                Log::info("No files need to be updated.");
            }

            // 7. Boxで削除されたファイルをDBから削除
            $boxFileIds = $boxFilesById->keys()->all();
            $dbFileIds = $dbFiles->keys()->all();
            $deletedIds = array_diff($dbFileIds, $boxFileIds);
            if (!empty($deletedIds)) {
                Log::info("Deleting " . count($deletedIds) . " records from database...");
                ModelFileCache::whereIn('box_file_id', $deletedIds)->delete();
                Log::info("Deletion complete.");
            }
        } catch (Exception $e) {
            Log::error("Box Sync Failed: " . $e->getMessage(), ['exception' => $e]);
        }

        Log::info("Sync process finished. Returning cached data.");
        return $this->getCachedModelData($projectFolderId);
    }


    /**
     * 【新規・ヘルパー】フロントエンド用にデータを整形する
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
     * 【新規・ヘルパー】Boxからファイルリストを全て取得（更新日時も含む）
     */
    private function fetchFullBoxFileList($projectFolderId)
    {
        $accessToken = session('access_token');
        $client = new Client(['verify' => false]);
        $header = ["Authorization" => "Bearer " . $accessToken];
        $allFolderItems = [];
        $offset = 0;
        $limit = 1000;
        do {
            Log::info("I am here fetchFullBoxFileList.....");

            // "modified_at"フィールドをリクエストに追加
            $requestURL = "https://api.box.com/2.0/folders/{$projectFolderId}/items?fields=id,name,modified_at&limit={$limit}&offset={$offset}";
            $response = $client->get($requestURL, ['headers' => $header]);
            $body = json_decode($response->getBody()->getContents());

            if (isset($body->entries)) {
                $allFolderItems = array_merge($allFolderItems, $body->entries);
                $offset += count($body->entries);
            } else {
                break;
            }
        } while ($offset < $body->total_count);

        return $allFolderItems;
    }


    private function fetchMultipleBoxFileContentsConcurrently($files)
    {
        $accessToken = session('access_token');
        $downloadedContents = [];

        $client = new Client(['verify' => false]); // ここではリトライハンドラなしのシンプルなクライアント
        $header = ["Authorization" => "Bearer " . $accessToken];

        $requests = function ($files) use ($header) {
            foreach ($files as $file) {
                $url = "https://api.box.com/2.0/files/{$file->id}/content";
                yield $file->id => new GuzzleRequest('GET', $url, $header);
            }
        };

        $pool = new Pool($client, $requests($files), [
            'concurrency' => 10,
            'fulfilled' => function ($response, $fileId) use (&$downloadedContents) {
                $downloadedContents[$fileId] = $response->getBody()->getContents();
            },
            'rejected' => function ($reason, $fileId) {
                // 【重要】失敗した理由をそのままスローする
                // これにより、呼び出し元のtry-catchでエラーを捕捉できる
                throw $reason;
            },
        ]);

        $promise = $pool->promise();

        // ここでwait()を呼ぶと、rejectedでスローされた例外がここで発生する
        $promise->wait();

        return $downloadedContents;
    }
