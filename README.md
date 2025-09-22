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
use Illuminate\Support\Facades\Session;
use Throwable;
use Exception;

class SyncBoxProject implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $projectFolderId;
    protected $accessToken;
    protected $refreshToken;
    protected $boxLoginTime;
    protected $urlPath;

    /**
     * 新しいジョブインスタンスの生成
     *
     * @param string $projectFolderId
     * @return void
     */
    public function __construct($projectFolderId, $access_token, $refresh_token, $box_login_time, $urlPath)
    {
        $this->projectFolderId = $projectFolderId;
        $this->accessToken = $access_token;
        $this->refreshToken = $refresh_token;
        $this->boxLoginTime = $box_login_time;
        $this->urlPath = $urlPath;
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
        Log::info("Job started for project: {$this->projectFolderId}");
        $dldwhModel = new DLDHWDataImportModel();

        try {
            // ジョブ開始時にDBから最新のトークン情報を取得
            // $tokens = $dldwhModel->getLatestBoxTokens();
            // if (!$tokens) {
            //     throw new \Exception("No valid tokens found in the database. Please log in again.");
            // }

            // $accessToken = $tokens->access_token;
            // $refreshToken = $tokens->refresh_token;
            // $loginTime = $tokens->login_time;

            // // 2. 予防的なトークンリフレッシュ
            // if (time() > $loginTime + (60 * 50)) {
            //     Log::info('Box token is about to expire. Refreshing...');
            //     $newTokens = $dldwhModel->refreshBoxToken($refreshToken, $this->urlPath);
            //     if (!$newTokens) {
            //         throw new \Exception("Failed to refresh Box token.");
            //     }
            //     // 変数を新しいトークンで上書き
            //     $accessToken = $newTokens['access_token'];
            //     $refreshToken = $newTokens['refresh_token'];
            // }

            // Log::info("Url JOb >>>>>" . $this->urlPath);

            // 3. モデルに有効なアクセストークンを渡して同期を実行
            Log::info('refresh_token In the Job' . $this->refreshToken);

            $filesUpdatedCount = $dldwhModel->syncAndGetModelData($this->projectFolderId, $this->accessToken, $this->refreshToken, $this->urlPath, $this->boxLoginTime);

            $status = ($filesUpdatedCount > 0) ? 'completed' : 'no_files_to_update';
            $this->saveJobLog($status);
            Log::info("Job completed for project {$this->projectFolderId} with status: {$status}");
        } catch (Throwable $e) {
            $this->failed($e);
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
