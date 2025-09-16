<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use App\Models\DLDHWDataImportModel;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Cache;
use Exception;

class SyncBoxProject implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $projectFolderId;
    protected $sessionData;

    /**
     * 新しいジョブインスタンスの生成
     *
     * @param string $projectFolderId
     * @param array $sessionData
     * @return void
     */
    public function __construct($projectFolderId, array $sessionData)
    {
        $this->projectFolderId = $projectFolderId;
        $this->sessionData = $sessionData;
    }

    /**
     * ジョブの実行
     *
     * このメソッドは、`php artisan queue:work`コマンドによって
     * バックグラウンドで実行されます。
     *
     * @return void
     */
    public function handle()
    {
        $jobStatusKey = "sync-status-{$this->projectFolderId}";
        Log::info("Job started for project: {$this->projectFolderId}");

        // ジョブの実行中にトークンを管理するためにセッションを模倣
        session($this->sessionData);

        try {
            // DLDHWDataImportModelの同期メソッドを呼び出す
            $dldwhModel = new DLDHWDataImportModel();
            $dldwhModel->syncAndGetModelData($this->projectFolderId);

            // 成功したらキャッシュを削除（または'completed'に更新）
            Cache::forget($jobStatusKey);
            
            Log::info("Successfully completed sync job for project: {$this->projectFolderId}");

        } catch (Exception $e) {
            // 失敗した場合もキャッシュを更新してフロントに知らせる
            Cache::put($jobStatusKey, 'failed', now()->addHours(12));
            Log::error("Sync job failed for project {$this->projectFolderId}: " . $e->getMessage(), ['exception' => $e]);
            
            // 例外を再スローしてジョブを失敗としてマークする
            // これにより、failed_jobsテーブルに記録される
            $this->fail($e);
        }
    }
}



<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use App\Jobs\SyncBoxProject; // 作成したジョブをインポート
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Cache;

class DLDWHDataObjectViewerController extends Controller
{
    public function objViewer()
    {
        return view('DLDWH.OBJViewer');
    }
    
    // ... getProjectList, getCategoryNameByElementId は変更なし ...

    /**
     * モデルデータを取得するためのメインAPI
     * 同期ジョブを投入し、キャッシュの状態とデータを返す
     */
    public function getModelData(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        if (empty($projectFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }

        $jobStatusKey = "sync-status-{$projectFolderId}";

        // Boxにログインしているかチェック
        if (session()->has('access_token') && !empty(session('access_token'))) {
            // 現在、このプロジェクトの同期ジョブが実行中でないかチェック
            if (!Cache::has($jobStatusKey)) {
                Log::info("Dispatching SyncBoxProject job for project: {$projectFolderId}");
                
                // ジョブ投入前に「実行中」の状態をキャッシュに保存（有効期限12時間）
                Cache::put($jobStatusKey, 'processing', now()->addHours(12));

                // ジョブに渡すセッションデータを準備
                $sessionData = [
                    'access_token' => session('access_token'),
                    'refresh_token' => session('refresh_token')
                ];

                // ジョブをキューに投入する
                SyncBoxProject::dispatch($projectFolderId, $sessionData);
            }
        }
        
        $dldwhModel = new DLDHWDataImportModel();
        $modelData = $dldwhModel->getCachedModelData($projectFolderId);

        // レスポンス構造を統一
        $responseData = [
            'sync_status' => Cache::get($jobStatusKey, 'completed'),
            'objMtlPairs' => [],
        ];

        if (isset($modelData['error'])) {
            $responseData['error'] = $modelData['error'];
        } else {
            $responseData['objMtlPairs'] = $modelData;
        }
        
        return response()->json($responseData);
    }

    /**
     * フロントエンドが同期状態を問い合わせるためのAPI
     */
    public function getSyncStatus(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        if (empty($projectFolderId)) {
            return response()->json(['sync_status' => 'error', 'message' => 'Folder ID is required.'], 400);
        }

        $jobStatusKey = "sync-status-{$projectFolderId}";
        $status = Cache::get($jobStatusKey, 'completed');

        return response()->json(['sync_status' => $status]);
    }
}






