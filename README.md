<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use App\Models\DLDHWDataImportModel;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Session;
use Illuminate\Support\Facades\DB; // DBファサードをインポート
use Throwable;

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
        Log::info("Job started for project: {$this->projectFolderId}");
        
        try {
            $this->checkBoxExpiryStatus();
            Session::put($this->sessionData);

            $dldwhModel = new DLDHWDataImportModel();
            // syncAndGetModelDataは更新されたファイル数を返すように修正済みと仮定
            $filesUpdatedCount = $dldwhModel->syncAndGetModelData($this->projectFolderId);

            $status = ($filesUpdatedCount > 0) ? 'completed' : 'no_files_to_update';
            $this->saveJobLog($status);
            
            Log::info("Successfully completed sync job for project: {$this->projectFolderId} with status: {$status}");

        } catch (Throwable $e) {
            $this->fail($e); // 失敗した場合はfailed()メソッドを呼び出す
        }
    }
    
    public function failed(Throwable $exception)
    {
        $this->saveJobLog('failed', $exception->getMessage());
        Log::error("Sync job has failed for project {$this->projectFolderId}: " . $exception->getMessage());
    }
    
    /**
     * job_logsテーブルに結果を保存する
     */
    private function saveJobLog($status, $message = null)
    {
        DB::connection('dldwh')->table('job_logs')->updateOrInsert(
            ['job_identifier' => $this->projectFolderId], // このキーで検索
            [ // 以下の内容で更新または作成
                'status' => $status,
                'message' => $message,
                'created_at' => now(),
                'updated_at' => now()
            ]
        );
    }
    
    // ... checkBoxExpiryStatus メソッドは変更なし ...
}
