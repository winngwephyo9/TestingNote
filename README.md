ALTER TABLE model_file_cache
ADD COLUMN `project_name` VARCHAR(255) NULL AFTER `project_box_id`;



<?php

namespace App\Models;

use App\Models\ModelFileCache;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use GuzzleHttp\Client;
// ... 他のuse文 ...
use Exception;

class DLDHWDataImportModel extends Model
{
    // ... getCategoryNameByElementId は変更なし ...
    
    /**
     * 【修正版】DBキャッシュから、保存されているプロジェクトのリストを取得する
     */
    public function getCachedProjectList()
    {
        // project_box_idとproject_nameでグループ化し、ユニークなプロジェクトリストを生成
        $projects = ModelFileCache::select('project_box_id', 'project_name')
            ->whereNotNull('project_name') // プロジェクト名がNULLでないものに限定
            ->distinct()
            ->get()
            ->map(function ($file) {
                return [
                    'id' => $file->project_box_id,
                    'name' => $file->project_name,
                ];
            });

        return $projects;
    }

    /**
     * 【修正版】同期時にプロジェクト名をDBに保存する
     */
    public function syncAndGetModelData($projectFolderId)
    {
        mb_internal_encoding('UTF-8');
        set_time_limit(0); 

        Log::info("Starting sync process for project: {$projectFolderId}");
        try {
            // =================================================================
            //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
            //
            //  **【重要】Boxからプロジェクト名（フォルダ名）も取得する**
            //
            $accessToken = session('access_token');
            if (empty($accessToken)) throw new Exception("Box access token is missing.");
            
            $client = new Client(['verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken];

            // フォルダ情報を取得してプロジェクト名を得る
            $folderInfoUrl = "https://api.box.com/2.0/folders/{$projectFolderId}?fields=name";
            $folderInfoResponse = $client->get($folderInfoUrl, ['headers' => $header]);
            $projectName = json_decode($folderInfoResponse->getBody()->getContents())->name;
            //
            //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
            // =================================================================

            $boxFiles = $this->fetchFullBoxFileList($projectFolderId);
            $boxFilesById = collect($boxFiles)->keyBy('id');
            $dbFiles = ModelFileCache::where('project_box_id', $projectFolderId)
                         ->get()->keyBy('box_file_id');
            
            // ... 更新が必要なファイルのリスト作成ロジック（変更なし） ...
            
            if (!empty($filesToUpdate)) {
                $updateChunks = array_chunk($filesToUpdate, 200);

                foreach ($updateChunks as $chunkIndex => $chunkOfFiles) {
                    $result = $this->fetchMultipleBoxFileContentsConcurrently($chunkOfFiles);
                    if ($result['status'] === 'error') throw new Exception($result['message']);
                    
                    $downloadedContents = $result['contents'];
                    $dataToUpsert = [];

                    foreach ($downloadedContents as $fileId => $content) {
                        $boxFile = $boxFilesById->get($fileId);
                        if ($boxFile) {
                            $dataToUpsert[] = [
                                'project_box_id' => $projectFolderId,
                                'project_name' => $projectName, // <-- 【重要】取得したプロジェクト名をセット
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
                        ModelFileCache::upsert(
                            $dataToUpsert, 
                            ['box_file_id'], 
                            // 【重要】更新対象カラムに project_name を追加
                            ['project_box_id', 'project_name', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at']
                        );
                    }
                    unset($downloadedContents, $dataToUpsert);
                }
            }
            // ... 削除処理などの残りのロジックは変更なし ...

        } catch (Exception $e) {
            Log::error("Box Sync Failed: " . $e->getMessage(), ['exception' => $e]);
            throw $e;
        }
        
        return true;
    }

    // ... その他のヘルパーメソッドは変更ありません ...
}
