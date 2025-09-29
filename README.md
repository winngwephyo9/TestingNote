<?php

namespace App\Models;

// ... use文 ...
use Illuminate\Support\Facades\Storage;
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Psr7\Request as GuzzleRequest;
use GuzzleHttp\RequestOptions;
use GuzzleHttp\Exception\ClientException;

class DLDHWDataImportModel extends Model
{
    // ... 他のメソッドは変更なし ...
    
    /**
     * 【最終修正版】ファイルをストリーミングでダウンロードし、直接ファイルに保存する
     * - getAdapter()エラーを修正
     * - LaravelのStorageファサードを全面的に活用
     */
    private function streamFilesWithGuzzle($files, $projectFolderId, $projectName, $accessToken, &$downloadedFilesMetadata = [])
    {
        $tokenExpired = false;
        $client = new Client(['verify' => false]);
        $header = ["Authorization" => "Bearer " . $accessToken];

        // =================================================================
        //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
        //
        //  **【重要】リクエストジェネレータの修正**
        //
        $requests = function ($files) use ($header, $projectFolderId) {
            foreach ($files as $file) {
                $url = "https://api.box.com/2.0/files/{$file['id']}/content";
                
                // 1. 保存先の相対パスを定義 (例: 'model_files/12345/67890_file.obj')
                $relativePath = "model_files/{$projectFolderId}/{$file['id']}_{$file['name']}";
                
                // 2. Storage::path()で、その相対パスに対するサーバー上の絶対パスを取得
                $absolutePath = Storage::disk('local')->path($relativePath);
                
                // 3. 親ディレクトリが存在するか確認し、なければStorageが自動作成
                $directoryPath = dirname($absolutePath);
                if (!Storage::disk('local')->exists("model_files/{$projectFolderId}")) {
                    Storage::disk('local')->makeDirectory("model_files/{$projectFolderId}");
                }
                
                // 【キーポイント】Guzzleの'sink'オプションには、この絶対パスを渡す
                yield $file['id'] => new GuzzleRequest('GET', $url, $header, null, ['sink' => $absolutePath]);
            }
        };
        //
        //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
        // =================================================================

        $pool = new Pool($client, $requests($files), [
            'concurrency' => 10,
            'fulfilled' => function ($response, $fileId) use (&$downloadedFilesMetadata, $projectFolderId, $projectName, $files) {
                $boxFile = collect($files)->firstWhere('id', $fileId);
                if ($boxFile) {
                    // DBに保存するのは、Storageのrootからの相対パス
                    $relativePath = "model_files/{$projectFolderId}/{$fileId}_{$boxFile['name']}";
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
}
