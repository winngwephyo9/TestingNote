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

class SyncBoxProject implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $projectFolderId;
    protected $sessionData;

    /**
     * 新しいジョブインスタンスの生成
     *
     * @param string $projectFolderId
     * @param array $sessionData (access_token, refresh_token, box_login_time, url_path)
     * @return void
     */
    public function __construct($projectFolderId, array $sessionData)
    {
        $this->projectFolderId = $projectFolderId;
        $this->sessionData = $sessionData;
    }

    /**
     * ジョブの実行
     */
    public function handle()
    {
        $jobStatusKey = "sync-status-{$this->projectFolderId}";
        Log::info("Job started for project: {$this->projectFolderId}");
        
        try {
            // 処理開始前にトークンの有効期限をチェック
            $this->checkBoxExpiryStatus();

            // 更新された可能性のあるトークンでセッションを上書き
            Session::put($this->sessionData);

            $dldwhModel = new DLDHWDataImportModel();
            $dldwhModel->syncAndGetModelData($this->projectFolderId);

            // 成功したらキャッシュを'completed'に更新
            Cache::put($jobStatusKey, 'completed', now()->addMinutes(5));
            Log::info("Successfully completed sync job for project: {$this->projectFolderId}");

        } catch (Throwable $e) {
            // 失敗した場合はキャッシュを'failed'に更新
            Cache::put($jobStatusKey, 'failed', now()->addHours(12));
            Log::error("Sync job failed for project {$this->projectFolderId}: " . $e->getMessage(), ['exception' => $e]);
            $this->fail($e);
        }
    }
    
    /**
     * ジョブ内でトークンの有効期限をチェックし、更新するメソッド
     */
    private function checkBoxExpiryStatus()
    {
        $loginTime = $this->sessionData['box_login_time'] ?? 0;
        $tokenExpiryTime = $loginTime + 3600; // 60分

        // トークンの有効期限が10分未満の場合
        if (time() > $tokenExpiryTime - 600) {
            Log::info('Box token is about to expire. Attempting to refresh.');
            
            $dldwhModel = new DLDHWDataImportModel();
            $newTokens = $dldwhModel->refreshBoxToken(
                $this->sessionData['refresh_token'],
                $this->sessionData['url_path']
            );
            
            if ($newTokens) {
                // リフレッシュに成功したら、このジョブが保持するトークン情報を更新
                $this->sessionData['access_token'] = $newTokens['access_token'];
                $this->sessionData['refresh_token'] = $newTokens['refresh_token'];
                $this->sessionData['box_login_time'] = time(); // ログイン時刻も現在時刻に更新
                Log::info('Tokens updated within the job instance.');
            } else {
                // リフレッシュに失敗した場合、ジョブは続行不可能なので例外をスロー
                throw new Exception("Failed to refresh Box token. Cannot continue sync job.");
            }
        }
    }

    /**
     * ジョブが最終的に失敗したときに呼び出される
     */
    public function failed(Throwable $exception)
    {
        $jobStatusKey = "sync-status-{$this->projectFolderId}";
        Cache::put($jobStatusKey, 'failed', now()->addHours(12));
        Log::error("Sync job has failed definitively for project {$this->projectFolderId}. Error: " . $exception->getMessage());
    }
}
