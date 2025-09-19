<?php

namespace App\Models;

use App\Models\ModelFileCache;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Http;
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Exception\ClientException;
use Illuminate\Support\Carbon;
use Exception;
use Throwable;

class DLDHWDataImportModel extends Model
{
    // ... getCategoryNameByElementId, getCachedProjectList, getCachedModelData, formatDataForFrontend は変更なし ...
    
    /**
     * 【最終版】BoxとDBを同期するメインロジック
     * @return int 更新されたファイル数
     */
    public function syncAndGetModelData($projectFolderId)
    {
        mb_internal_encoding('UTF-8');
        set_time_limit(0); 

        try {
            $accessToken = session('access_token');
            if (empty($accessToken)) throw new Exception("Box access token is missing.");
            
            $client = new Client(['verify' => false]);
            $projectName = $this->fetchProjectName($projectFolderId, $client);
            
            $boxFiles = $this->fetchFullBoxFileList($projectFolderId, $client);
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
            $updateChunks = array_chunk($filesToUpdate, 200);

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
                            'created_at' => now(), 'updated_at' => now(),
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
     * 【新規】ダウンロードとトークンリフレッシュを自動で処理するラッパー
     */
    private function fetchContentsWithRefresh(array $filesToDownload)
    {
        $maxRetries = 2;
        for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
            $result = $this->fetchMultipleBoxFileContentsConcurrently($filesToDownload);

            if ($result['status'] === 'success') {
                return $result['contents']; // 成功したらすぐに返す
            }
            
            if ($result['status'] === 'token_expired') {
                Log::info("Token expired. Attempting refresh (Attempt {$attempt}/{$maxRetries})");
                $newTokens = $this->refreshBoxToken(session('refresh_token'));
                if (!$newTokens) {
                    throw new Exception("Failed to refresh Box token after expiration.");
                }
                // セッションを更新して、次のループで再試行
                continue;
            }
            
            // その他のエラー
            throw new Exception($result['message'] ?? 'Unknown download error.');
        }

        throw new Exception("Failed to download files after {$maxRetries} refresh attempts.");
    }
    
    /**
     * 【リファクタリング版】並列ダウンロード関数
     */
    private function fetchMultipleBoxFileContentsConcurrently(array $files)
    {
        $downloadedContents = [];
        $tokenExpired = false;
        
        $client = new Client(['verify' => false]);
        $header = ["Authorization" => "Bearer " . session('access_token')];
        
        $requests = function ($files) use ($header) { /* ... 変更なし ... */ };
        $pool = new Pool($client, $requests($files), [
            'concurrency' => 10,
            'fulfilled' => function ($response, $fileId) use (&$downloadedContents) { /* ... */ },
            'rejected' => function ($reason, $fileId) use (&$tokenExpired) {
                if ($reason instanceof ClientException && $reason->getResponse()->getStatusCode() == 401) {
                    $tokenExpired = true;
                } else {
                    Log::error("Failed to download file ID {$fileId}: " . $reason->getMessage());
                }
            }
        ]);
        $pool->promise()->wait();

        if ($tokenExpired) {
            return ['status' => 'token_expired', 'contents' => $downloadedContents];
        }

        return ['status' => 'success', 'contents' => $downloadedContents];
    }
    
    /**
     * 【リファクタリング版】トークンリフレッシュ関数
     */
    public function refreshBoxToken($refreshToken)
    {
        // ... (以前の回答と同じロジック、urlPathは不要) ...
        // 成功時にDBとセッションの両方を更新する
    }

    // ... DB操作ヘルパーメソッド (getLatestBoxTokens, saveBoxTokens, jobLogsUpdateOrInsertなど) ...
}
