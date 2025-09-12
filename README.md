public function syncAndGetModelData($projectFolderId)
    {
        // =================================================================
        //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
        //
        //  **【重要】この関数の実行時間制限を無効にする**
        //  多数のファイルをダウンロードするため、120秒を超える可能性がある
        //
        set_time_limit(0); 
        //
        //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
        // =================================================================
        Log::info("I am here syncAndGetModelData.....");

        try {
            // 1. Box APIから最新のファイルリストを取得
            $boxFiles = $this->fetchFullBoxFileList($projectFolderId);
            $boxFilesById = collect($boxFiles)->keyBy('id');

            // 2. DBから現在のファイルリストを取得
            $dbFiles = ModelFileCache::where('project_box_id', $projectFolderId)
                         ->get()
                         ->keyBy('box_file_id');
            
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
                Log::info("Starting download of " . count($filesToUpdate) . " files...");
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
                
                // 6. 1回のクエリで全てのデータを挿入・更新する
                if (!empty($dataToUpsert)) {
                    Log::info("Upserting " . count($dataToUpsert) . " records into database...");
                    ModelFileCache::upsert(
                        $dataToUpsert,
                        ['box_file_id'],
                        ['project_box_id', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at']
                    );
                    Log::info("Upsert complete.");
                }
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
            Log::error("Box Sync Failed: " . $e->getMessage());
        }
        
        Log::info("Sync process finished. Returning cached data.");
        return $this->getCachedModelData($projectFolderId);
    }
