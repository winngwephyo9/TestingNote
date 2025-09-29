<?php

// ... use文 ...

class DLDHWDataImportModel extends Model
{
    // ... 他のメソッドは変更なし ...
    
    /**
     * 【最終修正版】ファイル名をハッシュ化してパス長の制限を回避する
     */
    private function streamFilesWithGuzzle($files, $projectFolderId, $projectName, $accessToken, &$downloadedFilesMetadata = [])
    {
        $tokenExpired = false;
        // ... (Guzzle ClientとHeaderの準備) ...

        $requests = function ($files) use ($header, $projectFolderId, $storageRoot) {
            foreach ($files as $file) {
                $url = "https://api.box.com/2.0/files/{$file['id']}/content";
                
                // =================================================================
                //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
                //
                //  **【重要】ファイル名をハッシュ化して、パスを短くする**
                //
                // 1. 元のファイル名から拡張子を取得
                $extension = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
                
                // 2. 元のファイル名をMD5でハッシュ化（常に32文字になる）
                $hashedName = md5($file['name']);
                
                // 3. 短くて確実なファイル名を生成
                $safeFileName = "{$file['id']}_{$hashedName}.{$extension}";
                //
                //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
                // =================================================================
                
                $relativePathForDir = "model_files" . DIRECTORY_SEPARATOR . $projectFolderId;
                $absoluteDirectoryPath = $storageRoot . DIRECTORY_SEPARATOR . $relativePathForDir;
                if (!file_exists($absoluteDirectoryPath)) {
                    mkdir($absoluteDirectoryPath, 0775, true);
                }
                
                // Guzzleの'sink'に渡す、最終的なファイルの絶対パス
                $absoluteFilePath = $absoluteDirectoryPath . DIRECTORY_SEPARATOR . $safeFileName;

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
                    $hashedName = md5($boxFile['name']);
                    $safeFileName = "{$fileId}_{$hashedName}.{$extension}";
                    
                    $relativePath = "model_files/{$projectFolderId}/{$safeFileName}";
                    
                    $downloadedFilesMetadata[] = [
                        // ... (他のメタデータは変更なし) ...
                        'file_path' => $relativePath, // ハッシュ化したパスを保存
                    ];
                }
            },
            'rejected' => function ($reason, $fileId) use (&$tokenExpired) {
                // ... (rejectedのロジックは変更なし) ...
            }
        ]);
        
        $pool->promise()->wait();
        
        if ($tokenExpired) return ['status' => 'token_expired'];
        return ['status' => 'success', 'metadata' => $downloadedFilesMetadata];
    }
}
