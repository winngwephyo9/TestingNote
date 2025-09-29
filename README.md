/**
     * 【OBJ】Guzzle Poolに渡すリクエストの形式を修正
     */
    private function streamFilesWithGuzzle(array $files, $projectName)
    {
        $tokenExpired = false;
        $projectFolderId = $this->projectFolderId;
        $client = new Client(['verify' => false]);
        $header = ["Authorization" => "Bearer " . $this->accessToken];
        $storageRoot = config('filesystems.disks.local.root');
        $requests = function ($files) use ($header, $projectFolderId, $storageRoot) {
            foreach ($files as $file) {
                $url = "https://api.box.com/2.0/files/{$file['id']}/content";
                // 1. 元のファイル名から拡張子を取得
                $extension = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
                Log::info("File Name>>>>>>" . $file['name']);
                // 2. 元のファイル名をMD5でハッシュ化（常に32文字になる）
                $hashedName = md5($file['name']);
                Log::info("Hashed Name>>>>>>" . $hashedName);

                // 3. 短くて確実なファイル名を生成
                $safeFileName = "{$file['id']}_{$hashedName}.{$extension}";
                Log::info("safeFileName >>>>>>" . $safeFileName);

                $relativePathForDir = "model_files" . DIRECTORY_SEPARATOR . $projectFolderId;
                Log::info("relativePathForDir >>>>>>" . $relativePathForDir);

                $absoluteDirectoryPath = $storageRoot . DIRECTORY_SEPARATOR . $relativePathForDir;
                Log::info("absoluteDirectoryPath >>>>>>" . $absoluteDirectoryPath);

                if (!file_exists($absoluteDirectoryPath)) {
                    mkdir($absoluteDirectoryPath, 0775, true);
                }

                // Guzzleの'sink'に渡す、最終的なファイルの絶対パス
                $absoluteFilePath = $absoluteDirectoryPath . DIRECTORY_SEPARATOR . $safeFileName;
                Log::info("absoluteFilePath >>>>>>" . $absoluteFilePath);

                yield $file['id'] => new \GuzzleHttp\Psr7\Request('GET', $url, $header, null, ['sink' => $absoluteFilePath]);
            }
        };

        $pool = new \GuzzleHttp\Pool($client, $requests($files), [
            'concurrency' => 10,
            'fulfilled' => function ($response, $fileId) use (&$downloadedFilesMetadata, $projectFolderId, $projectName, $files) {
                $boxFile = collect($files)->firstWhere('id', $fileId);
                if ($boxFile) {
                    // DBに保存するパスも、ハッシュ化したファイル名を使う
                    $extension = strtolower(pathinfo($boxFile['name'], PATHINFO_EXTENSION));
                    Log::info("extension >>>>>>" . $extension);

                    $hashedName = md5($boxFile['name']);
                    Log::info("hashedName >>>>>>" . $hashedName);

                    $safeFileName = "{$fileId}_{$hashedName}.{$extension}";
                    Log::info("safeFileName >>>>>>" . $safeFileName);


                    $relativePath = "model_files/{$projectFolderId}/{$safeFileName}";
                    Log::info("relativePath >>>>>>" . $relativePath);


                    $downloadedFilesMetadata[] = [
                        'project_box_id' => $projectFolderId,
                        'project_name' => $projectName,
                        'file_name' => $boxFile['name'],
                        'base_name' => pathinfo($boxFile['name'], PATHINFO_FILENAME),
                        'box_file_id' => $fileId,
                        'file_type' => strtolower(pathinfo($boxFile['name'], PATHINFO_EXTENSION)),
                        'file_path' => $relativePath, // 相対パスを保存
                        'box_modified_at' => \Illuminate\Support\Carbon::parse($boxFile['modified_at'])->setTimezone('Asia/Tokyo'),
                        'created_at' => now(),
                        'updated_at' => now(),
                    ];
                }
            },
            'rejected' => function ($reason, $fileId) use (&$tokenExpired) {
                if ($reason instanceof ClientException && $reason->getResponse()->getStatusCode() == 401) {
                    $tokenExpired = true;
                } else {
                    Log::error("Failed to stream file ID {$fileId}: " . $reason->getMessage());
                }
            }
        ]);

        $pool->promise()->wait();

        if ($tokenExpired) return ['status' => 'token_expired'];
        return ['status' => 'success', 'metadata' => $downloadedFilesMetadata];
    }


    [2025-09-29 18:14:52] local.INFO: File Name>>>>>>240627_BIKEN15号棟_2022_非一体型の踊り場_17521225_20250904.mtl  
[2025-09-29 18:14:52] local.INFO: Hashed Name>>>>>>c0f58661b604c7b6ff053510736de392  
[2025-09-29 18:14:52] local.INFO: safeFileName >>>>>>1988055698212_c0f58661b604c7b6ff053510736de392.mtl  
[2025-09-29 18:14:52] local.INFO: relativePathForDir >>>>>>model_files\341439813396  
[2025-09-29 18:14:52] local.INFO: absoluteDirectoryPath >>>>>>C:\xampp\htdocs\CCC\ccc\storage\app\model_files\341439813396  
[2025-09-29 18:14:52] local.INFO: absoluteFilePath >>>>>>C:\xampp\htdocs\CCC\ccc\storage\app\model_files\341439813396\1988055698212_c0f58661b604c7b6ff053510736de392.mtl  

