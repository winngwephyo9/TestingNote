<?php

namespace App\Models;

use App\Models\ModelFileCache;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Psr7\Request as GuzzleRequest;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Middleware;
use GuzzleHttp\Psr7\Response;
use Exception;

class DLDHWDataImportModel extends Model
{
    /**
     * **【最終修正版】プレースホルダ上限と文字化け対策**
     */
    public function syncAndGetModelData($projectFolderId)
    {
        // PHPの内部エンコーディングをUTF-8に設定（文字化け対策）
        mb_internal_encoding('UTF-8');
        // 多数のファイルをダウンロードするため、実行時間制限を無効化
        set_time_limit(0); 

        Log::info("Starting sync process for project: {$projectFolderId}");

        try {
            // 1. Boxから最新のファイルリストを取得
            $boxFiles = $this->fetchFullBoxFileList($projectFolderId);
            $boxFilesById = collect($boxFiles)->keyBy('id');

            // 2. DBから現在のファイルリストを取得
            $dbFiles = ModelFileCache::where('project_box_id', $projectFolderId)
                         ->get()->keyBy('box_file_id');
            
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
                Log::info("Found " . count($filesToUpdate) . " files to update/insert. Starting download...");
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
                
                // 6. データを500件ずつのチャンクに分割してUpsertする（プレースホルダ上限エラー回避）
                if (!empty($dataToUpsert)) {
                    $chunks = array_chunk($dataToUpsert, 500); 

                    Log::info("Upserting " . count($dataToUpsert) . " records in " . count($chunks) . " chunks...");
                    foreach ($chunks as $index => $chunk) {
                        ModelFileCache::upsert(
                            $chunk,
                            ['box_file_id'], // 一意キー
                            ['project_box_id', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at'] // 更新対象カラム
                        );
                        Log::info("Upserted chunk " . ($index + 1) . "/" . count($chunks) . " (" . count($chunk) . " records).");
                    }
                    Log::info("Upsert complete.");
                }
            } else {
                Log::info("No files need to be updated.");
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
            Log::error("Box Sync Failed: " . $e->getMessage(), ['exception' => $e]);
        }
        
        Log::info("Sync process finished. Returning cached data.");
        return $this->getCachedModelData($projectFolderId);
    }

    // ... その他のヘルパー関数は変更ありません ...
}
