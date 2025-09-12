<?php

namespace App\Models;

// ... use文は前回と同じ ...
use Illuminate-Support-Facades-Log;
use GuzzleHttp-Client;
use GuzzleHttp-Pool;
use GuzzleHttp-Psr7-Request as GuzzleRequest;
use GuzzleHttp-Exception-ClientException; // ClientExceptionをインポート

class DLDHWDataImportModel extends Model
{
    // ... getCategoryNameByElementId, getCachedModelData は変更なし ...
    
    public function syncAndGetModelData($projectFolderId)
    {
        set_time_limit(0); 
        Log::info("I am here syncAndGetModelData.....");
        try {
            // ... ファイルリスト取得、更新リスト作成のロジックは変更なし ...

            if (!empty($filesToUpdate)) {
                Log::info("Starting download of " . count($filesToUpdate) . " files...");
                
                // 【重要】ダウンロード処理をtry-catchで囲む
                try {
                    $downloadedContents = $this->fetchMultipleBoxFileContentsConcurrently($filesToUpdate);
                    Log::info("Finished downloading " . count($downloadedContents) . " files.");

                    // ... DBへのUpsert処理は変更なし ...

                } catch (ClientException $e) {
                    // ClientException（400番台のエラー）をここでキャッチ
                    if ($e->getResponse()->getStatusCode() == 401) {
                        // 401 Unauthorizedエラーの場合
                        Log::error("Box token has expired during download process. Aborting sync.");
                        // ここで処理を中断し、既存のキャッシュを返す
                        return $this->getCachedModelData($projectFolderId);
                    } else {
                        // 401以外のクライアントエラー（404など）は再スロー
                        throw $e;
                    }
                }
            }
            
            // ... 削除処理のロジックは変更なし ...

        } catch (Exception $e) {
            Log::error("Box Sync Failed: " . $e->getMessage());
        }
        
        Log::info("Sync process finished. Returning cached data.");
        return $this->getCachedModelData($projectFolderId);
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
    
    // ... その他のヘルパー関数は変更なし ...
}
