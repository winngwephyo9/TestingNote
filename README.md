<?php

namespace App\Models;

use App\Models\ModelFileCache;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Psr7\Request as GuzzleRequest;
use GuzzleHttp\Exception\ClientException; // ClientExceptionをインポート
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Middleware;
use GuzzleHttp\Psr7\Response;
use Exception;

class DLDHWDataImportModel extends Model
{
    // ... getCachedModelData, formatDataForFrontend, fetchFullBoxFileList は変更ありません ...
    public function getCachedModelData($projectFolderId) { /* ... */ }
    private function formatDataForFrontend($groupedFiles) { /* ... */ }
    private function fetchFullBoxFileList($projectFolderId) { /* ... */ }


    /**
     * **【最終修正版】トークン切れによるハングアップ対策を施した最終バージョン**
     */
    public function syncAndGetModelData($projectFolderId)
    {
        mb_internal_encoding('UTF-8');
        set_time_limit(0); 

        Log::info("Starting sync process for project: {$projectFolderId}");
        try {
            $boxFiles = $this->fetchFullBoxFileList($projectFolderId);
            $boxFilesById = collect($boxFiles)->keyBy('id');
            $dbFiles = ModelFileCache::where('project_box_id', $projectFolderId)
                         ->get()->keyBy('box_file_id');
            
            $filesToUpdate = [];
            foreach ($boxFiles as $boxFile) {
                $dbFile = $dbFiles->get($boxFile->id);
                if (!$dbFile || new \DateTime($dbFile->box_modified_at) < new \DateTime($boxFile->modified_at)) {
                    $filesToUpdate[] = $boxFile;
                }
            }
            
            if (!empty($filesToUpdate)) {
                Log::info("Found " . count($filesToUpdate) . " files to update/insert. Starting download...");
                
                // 【重要】ダウンロード処理の結果を$resultに格納
                $result = $this->fetchMultipleBoxFileContentsConcurrently($filesToUpdate);

                // トークン切れなどでダウンロードが中断されたかチェック
                if ($result['status'] === 'error') {
                    Log::error("Download process was aborted due to an error: " . $result['message']);
                    // 中断された場合は、同期を諦めて既存のキャッシュを返す
                    return $this->getCachedModelData($projectFolderId);
                }
                
                $downloadedContents = $result['contents'];
                Log::info("Finished downloading " . count($downloadedContents) . " files.");
                
                if (empty($downloadedContents)) {
                    Log::warning("No files were successfully downloaded. Skipping database update.");
                } else {
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

                    if (!empty($dataToUpsert)) {
                        $chunks = array_chunk($dataToUpsert, 500);
                        Log::info("Upserting " . count($dataToUpsert) . " records in " . count($chunks) . " chunks...");
                        foreach ($chunks as $index => $chunk) {
                            ModelFileCache::upsert($chunk, ['box_file_id'], ['project_box_id', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at']);
                            Log::info("Upserted chunk " . ($index + 1) . "/" . count($chunks) . " (" . count($chunk) . " records).");
                        }
                        Log::info("Upsert complete.");
                    }
                }
            } else {
                Log::info("No files need to be updated.");
            }
            
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
     * **【修正版】トークン切れに対応した堅牢な並列ダウンロード関数**
     */
    private function fetchMultipleBoxFileContentsConcurrently($files)
    {
        $accessToken = session('access_token');
        $downloadedContents = [];
        $tokenExpired = false; // トークン切れを検知するフラグ

        // レート制限(429)やサーバーエラー(5xx)に対応するリトライロジック
        $stack = HandlerStack::create();
        $stack->push(Middleware::retry(
            function ($retries, $request, $response = null, $exception = null) use (&$tokenExpired) {
                // 既にトークン切れが検知されていたら、一切リトライしない
                if ($tokenExpired) {
                    return false;
                }
                if ($retries >= 5) {
                    return false;
                }
                // 401エラーはリトライ対象外
                if ($response && in_array($response->getStatusCode(), [429, 502, 503, 504])) {
                    Log::warning("Retrying request due to status " . $response->getStatusCode() . ". Attempt: " . ($retries + 1));
                    return true;
                }
                return false;
            },
            function ($retries) {
                return (int)pow(2, $retries) * 1000 + rand(0, 100);
            }
        ));

        $client = new Client(['handler' => $stack, 'verify' => false]);
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
            'rejected' => function ($reason, $fileId) use (&$tokenExpired) {
                // rejectedハンドラ内でエラーの種類を特定
                if ($reason instanceof ClientException && $reason->getResponse()->getStatusCode() == 401) {
                    // 401エラーの場合、フラグを立てて他の処理に知らせる
                    $tokenExpired = true;
                    Log::error("Box token expired (401 Unauthorized). Aborting further downloads.");
                } else {
                    // 401以外のエラー
                    Log::error("Failed to download content for file ID {$fileId}: " . $reason->getMessage());
                }
            },
        ]);

        $promise = $pool->promise();
        
        try {
            // プールを実行
            $promise->wait();
        } catch(Exception $e) {
            // rejectedでスローされなかった予期せぬエラーをキャッチ
            Log::error("An unexpected error occurred during the Guzzle pool execution: " . $e->getMessage());
             return ['status' => 'error', 'message' => $e->getMessage(), 'contents' => []];
        }

        // wait()が終了した後、トークン切れフラグをチェック
        if ($tokenExpired) {
            // トークン切れが発生していたら、エラーを返す
            return ['status' => 'error', 'message' => 'Box token expired during the process.', 'contents' => $downloadedContents];
        }

        // 正常終了
        return ['status' => 'success', 'contents' => $downloadedContents];
    }
}
