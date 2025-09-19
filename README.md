<?php

namespace App\Jobs;

// ... use文 ...
use App\Models\DLDHWDataImportModel;
use Throwable;

class SyncBoxProject implements ShouldQueue
{
    protected $projectFolderId;

    public function __construct($projectFolderId)
    {
        $this->projectFolderId = $projectFolderId;
    }

    public function handle()
    {
        Log::info("Job started for project: {$this->projectFolderId}");
        $dldwhModel = new DLDHWDataImportModel();
        
        try {
            // ジョブ開始時にDBから最新のトークン情報を取得
            $latestTokens = $dldwhModel->getLatestBoxTokens();
            if (!$latestTokens) {
                throw new Exception("No valid tokens found in the database. Please log in again.");
            }
            session([
                'access_token' => $latestTokens->access_token,
                'refresh_token' => $latestTokens->refresh_token,
            ]);
            
            $filesUpdatedCount = $dldwhModel->syncAndGetModelData($this->projectFolderId);
            $status = ($filesUpdatedCount > 0) ? 'completed' : 'no_files_to_update';
            $this->saveJobLog($status);
            Log::info("Job completed for project {$this->projectFolderId} with status: {$status}");

        } catch (Throwable $e) {
            $this->fail($e);
        }
    }
    
    public function failed(Throwable $exception)
    {
        $this->saveJobLog('failed', $exception->getMessage());
        Log::error("Job failed for project {$this->projectFolderId}: " . $exception->getMessage());
    }
    
    private function saveJobLog($status, $message = null)
    {
        $model = new DLDHWDataImportModel();
        $model->jobLogsUpdateOrInsert($this->projectFolderId, $status, $message);
    }
}
