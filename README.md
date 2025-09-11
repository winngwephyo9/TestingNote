<?php

namespace App\Models;

use App\Models\ModelFileCache; // 必ずインポートする
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
        // ModelFileCacheが自動的に'dldwh'接続を使用する
        $cachedFiles = ModelFileCache::where('project_box_id', $projectFolderId)
                         ->get()
                         ->groupBy('base_name');

        if ($cachedFiles->isEmpty()) {
            return ['error' => 'No cached model found. Please log in to Box to sync data for the first time.'];
        }
        
        return $this->formatDataForFrontend($cachedFiles);
    }

    public function syncAndGetModelData($projectFolderId)
    {
        try {
            $boxFiles = $this->fetchFullBoxFileList($projectFolderId);
            $boxFilesById = collect($boxFiles)->keyBy('id');

            // ModelFileCacheが自動的に'dldwh'接続を使用する
            $dbFiles = ModelFileCache::where('project_box_id', $projectFolderId)
                         ->get()
                         ->keyBy('box_file_id');
            
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
                    // ModelFileCacheが自動的に'dldwh'接続でupsertを実行する
                    ModelFileCache::upsert(
                        $dataToUpsert,
                        ['box_file_id'],
                        ['project_box_id', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at']
                    );
                }
            }
            
            $boxFileIds = $boxFilesById->keys()->all();
            $dbFileIds = $dbFiles->keys()->all();
            $deletedIds = array_diff($dbFileIds, $boxFileIds);
            if (!empty($deletedIds)) {
                // ModelFileCacheが自動的に'dldwh'接続でdeleteを実行する
                ModelFileCache::whereIn('box_file_id', $deletedIds)->delete();
            }

        } catch (Exception $e) {
            Log::error("Box Sync Failed: " . $e->getMessage());
        }
        
        return $this->getCachedModelData($projectFolderId);
    }
    
    // ... formatDataForFrontend, fetchFullBoxFileList, fetchMultipleBoxFileContentsConcurrently は変更ありません ...
}


<?php

namespace App.Models;

use Illuminate\Database\Eloquent\Model;

class ModelFileCache extends Model
{
    /**
     * このモデルが使用するデータベース接続名。
     *
     * @var string
     */
    protected $connection = 'dldwh'; // <-- この行を追加！

    /**
     * このモデルが関連付けられるテーブル名。
     *
     * @var string
     */
    protected $table = 'model_file_cache';

    /**
     * マスアサインメントから保護する属性。
     *
     * @var array
     */
    protected $guarded = ['id'];

    /**
     * モデルの日付として扱う属性。
     *
     * @var array
     */
    protected $dates = [
        'box_modified_at',
    ];
    
    /**
     * Eloquentのタイムスタンプを有効にする。
     *
     * @var bool
     */
    public $timestamps = true;
}




