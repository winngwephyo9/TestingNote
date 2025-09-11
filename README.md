<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

/**
 * Class ModelFileCache
 *
 * @package App\Models
 * @property int $id
 * @property string $project_box_id
 * @property string $file_name
 * @property string $base_name
 * @property string $box_file_id
 * @property string $file_type
 * @property string|null $content
 * @property \Illuminate\Support\Carbon $box_modified_at
 * @property \Illuminate\Support\Carbon|null $created_at
 * @property \Illuminate\Support\Carbon|null $updated_at
 */
class ModelFileCache extends Model
{
    /**
     * このモデルが関連付けられるテーブル名。
     *
     * @var string
     */
    protected $table = 'model_file_cache';

    /**
     * マスアサインメント（一括代入）から保護する属性。
     *
     * $guardedプロパティに['id']を指定することで、id以外の全ての属性が
     * 一括で代入可能になります。upsertメソッドを安全に使うために重要です。
     *
     * @var array
     */
    protected $guarded = ['id'];

    /**
     * モデルの日付として扱う属性。
     *
     * この設定により、'box_modified_at' カラムの値が自動的に
     * Carbon（PHPの日時操作ライブラリ）インスタンスに変換されます。
     * これにより、日付の比較などが容易になります。
     *
     * @var array
     */
    protected $dates = [
        'box_modified_at',
    ];

    /**
     * Eloquentのタイムスタンプを無効にするかどうか。
     *
     * テーブルに created_at と updated_at カラムが存在するため、
     * この値は false のままにしておきます。
     *
     * @var bool
     */
    public $timestamps = true;
}


<?php

namespace App\Models;

// 作成したEloquentモデルをインポート
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
        // 取得部分もEloquentモデルを使うように変更（任意ですが推奨）
        $cachedFiles = ModelFileCache::where('project_box_id', $projectFolderId)
                         ->get()
                         ->groupBy('base_name');

        if ($cachedFiles->isEmpty()) {
            return ['error' => 'No cached model found. Please log in to Box to sync data for the first time.'];
        }
        
        return $this->formatDataForFrontend($cachedFiles);
    }

    /**
     * **【最終修正版】バルクUPSERTでデータベースのタイムアウトを解決**
     */
    public function syncAndGetModelData($projectFolderId)
    {
        try {
            // 1. Box APIから最新のファイルリストを取得
            $boxFiles = $this->fetchFullBoxFileList($projectFolderId);
            $boxFilesById = collect($boxFiles)->keyBy('id');

            // 2. DBから現在のファイルリストを取得
            $dbFiles = ModelFileCache::where('project_box_id', $projectFolderId)
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

                // 5. **ここが重要：UPSERT用のデータ配列を作成**
                $dataToUpsert = [];
                foreach ($downloadedContents as $fileId => $content) {
                    $boxFile = $boxFilesById->get($fileId);
                    if ($boxFile) {
                        $dataToUpsert[] = [
                            'project_box_id' => $projectFolderId,
                            'file_name' => $boxFile->name,
                            'base_name' => pathinfo($boxFile->name, PATHINFO_FILENAME),
                            'box_file_id' => $fileId, // upsertの検索キー
                            'file_type' => strtolower(pathinfo($boxFile->name, PATHINFO_EXTENSION)),
                            'content' => $content,
                            'box_modified_at' => new \DateTime($boxFile->modified_at),
                            'created_at' => now(),
                            'updated_at' => now(),
                        ];
                    }
                }
                
                // 6. **1回のクエリで全てのデータを挿入・更新する**
                if (!empty($dataToUpsert)) {
                    ModelFileCache::upsert(
                        $dataToUpsert, // [1] 挿入・更新するデータの配列
                        ['box_file_id'], // [2] 一意のキー（このカラムで重複をチェックする）
                        // [3] 重複があった場合に更新するカラムのリスト
                        ['project_box_id', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at']
                    );
                }
            }
            
            // 7. Boxで削除されたファイルをDBから削除
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
    
    // ... formatDataForFrontend, fetchFullBoxFileList, fetchMultipleBoxFileContentsConcurrently は変更ありません ...
}



