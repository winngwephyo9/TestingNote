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

    public function __construct($projectFolderId, array $sessionData)
    {
        $this->projectFolderId = $projectFolderId;
        $this->sessionData = $sessionData;
    }

    public function handle()
    {
        $jobStatusKey = "sync-status-{$this->projectFolderId}";
        Log::info("Job started for project: {$this->projectFolderId}");
        session($this->sessionData);

        try {
            $dldwhModel = new DLDHWDataImportModel();
            $dldwhModel->syncAndGetModelData($this->projectFolderId);

            // =================================================================
            //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
            //
            //  **【重要】完了状態を短い有効期限付きでキャッシュに保存**
            //  5分間は「完了済み」として扱うことで、UIからの連続ディスパッチを防ぐ
            //
            Cache::put($jobStatusKey, 'completed', now()->addMinutes(5));
            //
            //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
            // =================================================================
            
            Log::info("Successfully completed sync job for project: {$this->projectFolderId}");

        } catch (Exception $e) {
            Cache::put($jobStatusKey, 'failed', now()->addHours(12));
            Log::error("Sync job failed for project {$this->projectFolderId}: " . $e->getMessage(), ['exception' => $e]);
            $this->fail($e);
        }
    }
}



<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use App\Jobs\SyncBoxProject;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Cache;

class DLDWHDataObjectViewerController extends Controller
{
    public function getModelData(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        if (empty($projectFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }

        $jobStatusKey = "sync-status-{$projectFolderId}";
        $currentStatus = Cache::get($jobStatusKey);

        if (session()->has('access_token') && !empty(session('access_token'))) {
            
            // =================================================================
            //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
            //
            //  **【重要】ディスパッチ条件の修正**
            //
            //  現在処理中でもなく、かつ、直近で完了してもいない場合にのみディスパッチする
            //
            if ($currentStatus !== 'processing' && $currentStatus !== 'completed') {
                Log::info("Dispatching SyncBoxProject job for project: {$projectFolderId}. Previous status was: " . ($currentStatus ?? 'null'));
                
                Cache::put($jobStatusKey, 'processing', now()->addHours(12));
                $sessionData = [
                    'access_token' => session('access_token'),
                    'refresh_token' => session('refresh_token')
                ];
                SyncBoxProject::dispatch($projectFolderId, $sessionData);
            } else {
                 Log::info("Sync job for project {$projectFolderId} is already {$currentStatus}. Skipping dispatch.");
            }
            //
            //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
            // =================================================================
        }
        
        $dldwhModel = new DLDHWDataImportModel();
        $modelData = $dldwhModel->getCachedModelData($projectFolderId);

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
    // ... 他のメソッドは変更なし ...
}
