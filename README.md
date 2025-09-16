<?php

namespace App\Models;

use App\Models\ModelFileCache;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Promise\EachPromise;
use GuzzleHttp\Psr7\Request as GuzzleRequest;
use GuzzleHttp\Exception\ClientException;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Middleware;
use GuzzleHttp\Psr7\Response;
use Illuminate\Support\Carbon;
use Exception;

class DLDHWDataImportModel extends Model
{
    /**
     * 要素リストテーブルからカテゴリ名などを取得する
     */
    public function getCategoryNameByElementId($WSCenID, $elementIds)
    {
        $tableName = "doc_0201要素リスト";
        try {
            $placeholders = implode(',', array_fill(0, count($elementIds), '?'));
            $query = "SELECT  要素_ID, カテゴリー名,ファミリ名, タイプ_ID
                      FROM {$tableName}
                      WHERE WSCenID = ? AND 要素_ID IN ({$placeholders})";

            $bindings = array_merge([$WSCenID], $elementIds);
            $result = DB::connection('dldwh')->select($query, $bindings);

            $keyedResult = [];
            foreach ($result as $row) {
                $keyedResult[$row->要素_ID] = $row;
            }
            return response()->json($keyedResult);
        } catch (Exception $e) {
            return "An error has occurred: " . $e->getMessage();
        }
    }
    
    /**
     * DBキャッシュからモデルデータを取得する
     */
    public function getCachedModelData($projectFolderId)
    {
        $cachedFiles = ModelFileCache::where('project_box_id', $projectFolderId)
            ->get()->groupBy('base_name');

        if ($cachedFiles->isEmpty()) {
            return ['error' => 'No cached model found. Please log in to Box to sync data for the first time.'];
        }

        return $this->formatDataForFrontend($cachedFiles);
    }

    /**
     * BoxとDBを同期するメインロジック（ジョブから呼び出される）
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
                if (!$dbFile) {
                    $filesToUpdate[] = $boxFile;
                    continue;
                }
                $boxModifiedAtJst = Carbon::parse($boxFile->modified_at)->setTimezone('Asia/Tokyo')->startOfSecond();
                $dbModifiedAtJst = $dbFile->box_modified_at->startOfSecond();
                if ($dbModifiedAtJst->lt($boxModifiedAtJst)) {
                    $filesToUpdate[] = $boxFile;
                }
            }
            
            if (!empty($filesToUpdate)) {
                Log::info("Found " . count($filesToUpdate) . " files to update/insert.");
                $updateChunks = array_chunk($filesToUpdate, 200);

                foreach ($updateChunks as $chunkIndex => $chunkOfFiles) {
                    Log::info("Processing chunk " . ($chunkIndex + 1) . "/" . count($updateChunks) . " (" . count($chunkOfFiles) . " files)...");
                    $result = $this->fetchMultipleBoxFileContentsConcurrently($chunkOfFiles);
                    if ($result['status'] === 'error') {
                        Log::error("Download aborted in chunk " . ($chunkIndex + 1) . ": " . $result['message']);
                        throw new Exception("Download process was aborted: " . $result['message']);
                    }
                    $downloadedContents = $result['contents'];

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
                                'box_modified_at' => Carbon::parse($boxFile->modified_at)->setTimezone('Asia/Tokyo'),
                                'created_at' => now(),
                                'updated_at' => now(),
                            ];
                        }
                    }
                    if (!empty($dataToUpsert)) {
                        ModelFileCache::upsert($dataToUpsert, ['box_file_id'], ['project_box_id', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at']);
                    }
                    unset($downloadedContents, $dataToUpsert);
                }
            } else {
                Log::info("No files need to be updated. Sync is complete.");
            }
            
            $boxFileIds = $boxFilesById->keys()->all();
            $dbFileIds = $dbFiles->keys()->all();
            $deletedIds = array_diff($dbFileIds, $boxFileIds);
            if (!empty($deletedIds)) {
                ModelFileCache::whereIn('box_file_id', $deletedIds)->delete();
            }
        } catch (Exception $e) {
            Log::error("Box Sync Failed: " . $e->getMessage(), ['exception' => $e]);
            throw $e; // ジョブに失敗を通知するために例外を再スロー
        }
        
        Log::info("Sync process finished for project: {$projectFolderId}");
        return true;
    }

    /**
     * フロントエンド用にデータを整形する
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
     * Boxからファイルリストを全て取得（更新日時も含む）
     */
    private function fetchFullBoxFileList($projectFolderId)
    {
        $accessToken = session('access_token');
        if (empty($accessToken)) throw new Exception("Box access token is missing from session.");
        
        $client = new Client(['verify' => false]);
        $header = ["Authorization" => "Bearer " . $accessToken];
        $allFolderItems = [];
        $offset = 0;
        $limit = 1000;
        do {
            $requestURL = "https://api.box.com/2.0/folders/{$projectFolderId}/items?fields=id,name,modified_at&limit={$limit}&offset={$offset}";
            $response = $client->get($requestURL, ['headers' => $header]);
            $body = json_decode($response->getBody()->getContents());
            if (isset($body->entries)) {
                $allFolderItems = array_merge($allFolderItems, $body->entries);
                $offset += count($body->entries);
            } else {
                break;
            }
        } while (isset($body->total_count) && $offset < $body->total_count);
        return $allFolderItems;
    }
    
    /**
     * トークン切れに対応した堅牢な並列ダウンロード関数
     */
    private function fetchMultipleBoxFileContentsConcurrently($files)
    {
        $downloadedContents = [];
        $maxRetries = 2; 
        $attempt = 1;

        while ($attempt <= $maxRetries) {
            $accessToken = session('access_token');
            if (empty($accessToken)) throw new Exception("No access token available for download.");

            $tokenExpiredInThisAttempt = false;
            $client = new Client(['verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken];
            
            $remainingFiles = array_filter($files, function ($file) use ($downloadedContents) {
                return !isset($downloadedContents[$file->id]);
            });
            if (empty($remainingFiles)) break;

            Log::info("Download attempt #{$attempt}. Remaining files: " . count($remainingFiles));

            $requests = function ($files) use ($header) {
                foreach ($files as $file) {
                    $url = "https://api.box.com/2.0/files/{$file->id}/content";
                    yield $file->id => new GuzzleRequest('GET', $url, $header);
                }
            };
            
            $pool = new Pool($client, $requests($remainingFiles), [
                'concurrency' => 10,
                'fulfilled' => function ($response, $fileId) use (&$downloadedContents) {
                    $downloadedContents[$fileId] = $response->getBody()->getContents();
                },
                'rejected' => function ($reason, $fileId) use (&$tokenExpiredInThisAttempt) {
                    if ($reason instanceof ClientException && $reason->getResponse()->getStatusCode() == 401) {
                        if (!$tokenExpiredInThisAttempt) {
                            Log::warning("Box token expired (401). Will attempt to refresh after this batch.");
                            $tokenExpiredInThisAttempt = true;
                        }
                    } else {
                        Log::error("Failed to download content for file ID {$fileId}: " . $reason->getMessage());
                    }
                }
            ]);

            $promise = $pool->promise();
            $promise->wait();

            if ($tokenExpiredInThisAttempt) {
                $newAccessToken = $this->refreshBoxToken();
                if (!$newAccessToken) {
                    Log::error("Failed to refresh Box token. Aborting sync process.");
                    return ['status' => 'error', 'message' => 'Failed to refresh token.', 'contents' => $downloadedContents];
                }
                $attempt++;
                continue;
            }
            break;
        }
        return ['status' => 'success', 'contents' => $downloadedContents];
    }
    
    /**
     * Boxのアクセストークンをリフレッシュするヘルパーメソッド
     */
    private function refreshBoxToken()
    {
        try {
            $refreshToken = session('refresh_token');
            if (!$refreshToken) {
                Log::error("No refresh token found in session.");
                return null;
            }
            $clientId = env('BOX_CLIENT_ID');
            $clientSecret = env('BOX_CLIENT_SECRET');
            if (!$clientId || !$clientSecret) {
                Log::error("BOX_CLIENT_ID or BOX_CLIENT_SECRET is not set in .env file.");
                return null;
            }
            $client = new Client(['verify' => false]);
            $response = $client->post('https://api.box.com/oauth2/token', [
                'form_params' => [
                    'grant_type' => 'refresh_token',
                    'refresh_token' => $refreshToken,
                    'client_id' => $clientId,
                    'client_secret' => $clientSecret,
                ]
            ]);
            $data = json_decode($response->getBody()->getContents());
            if (isset($data->access_token) && isset($data->refresh_token)) {
                session(['access_token' => $data->access_token]);
                session(['refresh_token' => $data->refresh_token]);
                Log::info("Successfully refreshed Box tokens.");
                return $data->access_token;
            } else {
                Log::error("Refresh token response did not contain new tokens.");
                return null;
            }
        } catch (Exception $e) {
            Log::error("Exception occurred while refreshing Box token: " . $e->getMessage());
            return null;
        }
    }
}
