<?php

namespace App\Jobs;

use App\Models\DLDHWDataImportModel;
// ...
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
            // 1. DBから最新のトークンを取得
            $tokens = $dldwhModel->getLatestBoxTokens();
            if (!$tokens) {
                throw new \Exception("No valid tokens found in the database. Please log in again.");
            }
            
            $accessToken = $tokens->access_token;
            $refreshToken = $tokens->refresh_token;
            $loginTime = $tokens->login_time;

            // 2. 予防的なトークンリフレッシュ
            if (time() > $loginTime + (60 * 50)) {
                Log::info('Box token is about to expire. Refreshing...');
                $newTokens = $dldwhModel->refreshBoxToken($refreshToken);
                if (!$newTokens) {
                    throw new \Exception("Failed to refresh Box token.");
                }
                // 変数を新しいトークンで上書き
                $accessToken = $newTokens['access_token'];
                $refreshToken = $newTokens['refresh_token'];
            }
            
            // 3. モデルに有効なアクセストークンを渡して同期を実行
            $filesUpdatedCount = $dldwhModel->syncAndGetModelData($this->projectFolderId, $accessToken);

            $status = ($filesUpdatedCount > 0) ? 'completed' : 'no_files_to_update';
            $this->saveJobLog($status);
            Log::info("Job completed for project {$this->projectFolderId} with status: {$status}");

        } catch (Throwable $e) {
            $this->fail($e);
        }
    }
    
    // ... failed(), saveJobLog() は変更なし ...
    // ... checkBoxExpiryStatus() は handle() に統合されたため不要 ...
}
