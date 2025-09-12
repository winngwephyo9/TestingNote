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
    // ... 既存の getCategoryNameByElementId メソッドなど ...

    public function getCachedModelData($projectFolderId)
    {
        // ... この関数は変更ありません ...
    }

    /**
     * **【最終修正版】プレースホルダ上限と文字化け対策**
     */
    public function syncAndGetModelData($projectFolderId)
    {
        // PHPの内部エンコーディングをUTF-8に設定（文字化け対策）
        mb_internal_encoding('UTF-8');
        set_time_limit(0); 

        try {
            // ... ファイルリスト取得、更新リスト作成のロジックは変更ありません ...
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
                $downloadedContents = $this->fetchMultipleBoxFileContentsConcurrently($filesToUpdate);

                $dataToUpsert = [];
                foreach ($downloadedContents as $fileId => $content) {
                    $boxFile = $boxFilesById->get($fileId);
                    if ($boxFile) {
                        $dataToUpsert[] = [
                            'project_box_id' => $projectFolderId,
                            'file_name' => $boxFile->name,
                            // base_nameもUTF-8として正しく扱われるように
                            'base_name' => mb_convert_encoding(pathinfo($boxFile->name, PATHINFO_FILENAME), 'UTF-8'),
                            'box_file_id' => $fileId,
                            'file_type' => strtolower(pathinfo($boxFile->name, PATHINFO_EXTENSION)),
                            'content' => $content,
                            'box_modified_at' => new \DateTime($boxFile->modified_at),
                            'created_at' => now(),
                            'updated_at' => now(),
                        ];
                    }
                }
                
                // =================================================================
                //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
                //
                //  **【重要】データをチャンクに分割してUpsertする**
                //
                if (!empty($dataToUpsert)) {
                    Log::info("Upserting " . count($dataToUpsert) . " records in chunks...");
                    
                    // データを1000件ずつのチャンクに分割
                    $chunks = array_chunk($dataToUpsert, 1000); 

                    foreach ($chunks as $chunk) {
                        ModelFileCache::upsert(
                            $chunk,
                            ['box_file_id'],
                            ['project_box_id', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at']
                        );
                        Log::info("Upserted a chunk of " . count($chunk) . " records.");
                    }
                    
                    Log::info("Upsert complete.");
                }
                //
                //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
                // =================================================================
            }
            
            // ... 削除処理のロジックは変更ありません ...
            $boxFileIds = $boxFilesById->keys()->all();
            $dbFileIds = $dbFiles->keys()->all();
            $deletedIds = array_diff($dbFileIds, $boxFileIds);
            if (!empty($deletedIds)) {
                ModelFileCache::whereIn('box_file_id', $deletedIds)->delete();
            }

        } catch (Exception $e) {
            Log::error("Box Sync Failed: " . $e->getMessage());
        }
        
        return $this->getCachedModelData($projectFolderId);
    }
    
    // ... 他のヘルパー関数は変更ありません ...
}
