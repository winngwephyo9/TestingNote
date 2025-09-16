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
    // ... getCachedModelData, formatDataForFrontend, fetchFullBoxFileList は変更ありません ...

    /**
     * **【最終修正版】タイムゾーンをJSTに統一して比較する**
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

                // =================================================================
                //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
                //
                //  **【重要】両方の日時をJSTに変換し、マイクロ秒を切り捨ててから比較する**
                //
                // Boxの日時をJSTに変換
                $boxModifiedAtJst = Carbon::parse($boxFile->modified_at)->setTimezone('Asia/Tokyo')->startOfSecond();
                
                // DBの日時は既にJSTとして解釈されているので、マイクロ秒を丸めるだけ
                $dbModifiedAtJst = $dbFile->box_modified_at->startOfSecond();

                if ($dbModifiedAtJst->lt($boxModifiedAtJst)) {
                    // Boxの日時(JST)の方が新しい場合のみ更新対象とする
                    Log::info("File needs update [{$boxFile->name}]. DB(JST): {$dbModifiedAtJst->toDateTimeString()}, Box(JST): {$boxModifiedAtJst->toDateTimeString()}");
                    $filesToUpdate[] = $boxFile;
                }
                //
                //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
                // =================================================================
            }
            
            if (!empty($filesToUpdate)) {
                Log::info("Found " . count($filesToUpdate) . " files to update/insert. Starting download...");
                $result = $this->fetchMultipleBoxFileContentsConcurrently($filesToUpdate);

                if ($result['status'] === 'error') {
                    Log::error("Download process was aborted due to an error: " . $result['message']);
                    return $this->getCachedModelData($projectFolderId);
                }
                
                $downloadedContents = $result['contents'];
                Log::info("Finished downloading " . count($downloadedContents) . " files.");
                
                if (!empty($downloadedContents)) {
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
                                // **【重要】DBに保存する際もJSTに変換する**
                                'box_modified_at' => Carbon::parse($boxFile->modified_at)->setTimezone('Asia/Tokyo'),
                                'created_at' => now(), // now()はデフォルトでJSTを返す
                                'updated_at' => now(),
                            ];
                        }
                    }
                    if (!empty($dataToUpsert)) {
                        $chunks = array_chunk($dataToUpsert, 500);
                        foreach ($chunks as $chunk) {
                            ModelFileCache::upsert($chunk, ['box_file_id'], ['project_box_id', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at']);
                        }
                    }
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
        }
        
        Log::info("Sync process finished. Returning cached data.");
        return $this->getCachedModelData($projectFolderId);
    }

    // ... その他のヘルパー関数は変更ありません ...
}
