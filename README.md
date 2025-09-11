<?php

namespace App\Models;

use Illuminate.Database\Eloquent\Model;
use Illuminate.Support\Facades\DB;
use Illuminate\Support\Facades\Log; // Logをインポート
use GuzzleHttp\Client;
use GuzzleHttp\Pool; // Poolをインポート
use GuzzleHttp\Psr7\Request as GuzzleRequest; // Requestをインポート
use GuzzleHttp\HandlerStack; // HandlerStackをインポート
use GuzzleHttp\Middleware; // Middlewareをインポート
use GuzzleHttp\Psr7\Response; // Responseをインポート
use Exception;

class DLDHWDataImportModel extends Model
{
    // ... 既存の getCategoryNameByElementId メソッドなど ...

    public function getCachedModelData($projectFolderId)
    {
        // ... この関数は変更ありません ...
    }

    /**
     * **【修正版】並列ダウンロードでタイムアウトを解決**
     * Boxにログインしている時に、BoxとDBを同期し、最新のモデルデータを取得する
     */
    public function syncAndGetModelData($projectFolderId)
    {
        try {
            // 1. Box APIから最新のファイルリストを取得
            $boxFiles = $this->fetchFullBoxFileList($projectFolderId);
            $boxFilesById = collect($boxFiles)->keyBy('id');

            // 2. DBから現在のファイルリストを取得
            $dbFiles = DB::table('model_file_cache')
                         ->where('project_box_id', $projectFolderId)
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
                $downloadedContents = $this->fetchMultipleBoxFileContentsConcurrently($filesToUpdate);

                // 5. ダウンロードしたコンテンツでDBを更新
                foreach ($downloadedContents as $fileId => $content) {
                    $boxFile = $boxFilesById->get($fileId);
                    if ($boxFile) {
                        DB::table('model_file_cache')->updateOrInsert(
                            ['box_file_id' => $fileId],
                            [
                                'project_box_id' => $projectFolderId,
                                'file_name' => $boxFile->name,
                                'base_name' => pathinfo($boxFile->name, PATHINFO_FILENAME),
                                'file_type' => strtolower(pathinfo($boxFile->name, PATHINFO_EXTENSION)),
                                'content' => $content,
                                'box_modified_at' => new \DateTime($boxFile->modified_at),
                                'created_at' => now(),
                                'updated_at' => now(),
                            ]
                        );
                    }
                }
            }
            
            // 6. Boxで削除されたファイルをDBから削除
            $boxFileIds = $boxFilesById->keys()->all();
            $dbFileIds = $dbFiles->keys()->all();
            $deletedIds = array_diff($dbFileIds, $boxFileIds);
            if (!empty($deletedIds)) {
                DB::table('model_file_cache')->whereIn('box_file_id', $deletedIds)->delete();
            }

        } catch (Exception $e) {
            Log::error("Box Sync Failed: " . $e->getMessage());
        }
        
        return $this->getCachedModelData($projectFolderId);
    }
    
    // ... formatDataForFrontend, fetchFullBoxFileList は変更ありません ...

    /**
     * 【新規・ヘルパー】複数のファイルコンテンツをGuzzle Poolで並列ダウンロードする
     */
    private function fetchMultipleBoxFileContentsConcurrently($files)
    {
        $accessToken = session('access_token');
        $downloadedContents = [];
        
        // --- 以前実装した、レート制限に対応するためのリトライ機能付きハンドラ ---
        $stack = HandlerStack::create();
        $stack->push(Middleware::retry(
            function ($retries, $request, $response = null, $exception = null) {
                if ($retries >= 5) return false;
                if ($response && in_array($response->getStatusCode(), [429, 500, 502, 503, 504])) {
                    Log::warning("Retrying request for content. Attempt: {$retries}");
                    return true;
                }
                return false;
            },
            function ($retries, $response) {
                if ($response->hasHeader('Retry-After')) {
                    return (int)$response->getHeader('Retry-After')[0] * 1000;
                }
                return (int)pow(2, $retries) * 1000 + rand(0, 100);
            }
        ));

        $client = new Client(['handler' => $stack, 'verify' => false]);
        $header = ["Authorization" => "Bearer " . $accessToken];

        // リクエストを生成するジェネレータ
        $requests = function ($files) use ($header) {
            foreach ($files as $file) {
                $url = "https://api.box.com/2.0/files/{$file->id}/content";
                yield $file->id => new GuzzleRequest('GET', $url, $header);
            }
        };

        // プールを作成
        $pool = new Pool($client, $requests($files), [
            'concurrency' => 10, // 同時接続数
            'fulfilled' => function ($response, $fileId) use (&$downloadedContents) {
                $downloadedContents[$fileId] = $response->getBody()->getContents();
            },
            'rejected' => function ($reason, $fileId) {
                Log::error("Failed to download content for file ID {$fileId}: " . $reason->getMessage());
            },
        ]);

        // プールを実行
        $promise = $pool->promise();
        $promise->wait();
        
        return $downloadedContents;
    }
}
