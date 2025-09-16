<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use App\Models\DLDHWDataImportModel; // モデルをインポート
use Illuminate\Support\Facades\Log;
use Exception;

class SyncBoxProject implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $projectFolderId;
    protected $initialAccessToken;
    protected $initialRefreshToken; // リフレッシュトークンを保持

    /**
     * 新しいジョブインスタンスの生成
     *
     * @param string $projectFolderId
     * @param string $accessToken
     * @param string $refreshToken
     * @return void
     */
    public function __construct($projectFolderId, $accessToken, $refreshToken)
    {
        $this->projectFolderId = $projectFolderId;
        $this->initialAccessToken = $accessToken;
        $this->initialRefreshToken = $refreshToken;
    }

    /**
     * ジョブの実行
     *
     * @return void
     */
    public function handle()
    {
        // このジョブ専用のモデルインスタンスを作成
        $dldwhModel = new DLDHWDataImportModel();

        // ジョブの実行中にトークンを管理するためのセッションを模倣
        // これにより、モデル内の既存のロジックを大きく変更せずに済む
        session(['access_token' => $this->initialAccessToken]);
        session(['refresh_token' => $this->initialRefreshToken]);

        try {
            // DLDHWDataImportModelの同期メソッドを呼び出す
            // このメソッドは、必要に応じてトークンをリフレッシュする機能を持つ必要がある
            $dldwhModel->syncAndGetModelData($this->projectFolderId);

            Log::info("Successfully completed sync job for project: {$this->projectFolderId}");

        } catch (Exception $e) {
            Log::error("Sync job failed for project {$this->projectFolderId}: " . $e->getMessage(), ['exception' => $e]);
            // ジョブを失敗としてマークし、再試行させる（任意）
            $this->fail($e);
        }
    }
}
