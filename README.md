<?php

namespace App\Models;

use App\Models\ModelFileCache;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Http;
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Exception\ClientException;
use Illuminate\Support\Carbon;
use Exception;
use Throwable;

class DLDHWDataImportModel extends Model
{
    // ... getCategoryNameByElementId, getCachedProjectList, etc. ...

    /**
     * 【最終版】BoxとDBを同期するメインロジック
     */
    public function syncAndGetModelData($projectFolderId, $accessToken, $refreshToken, $urlPath)
    {
        mb_internal_encoding('UTF-8');
        set_time_limit(0); 

        try {
            $client = new Client(['verify' => false]);
            $projectName = $this->fetchProjectName($projectFolderId, $accessToken, $client);
            $boxFiles = $this->fetchFullBoxFileList($projectFolderId, $accessToken, $client);
            $boxFilesById = collect($boxFiles)->keyBy('id');
            $dbFiles = ModelFileCache::where('project_box_id', $projectFolderId)->get()->keyBy('box_file_id');
            
            $filesToUpdate = [];
            foreach ($boxFiles as $boxFile) {
                $dbFile = $dbFiles->get($boxFile['id']);
                $boxModifiedAtJst = Carbon::parse($boxFile['modified_at'])->setTimezone('Asia/Tokyo')->startOfSecond();
                $dbModifiedAtJst = $dbFile ? $dbFile->box_modified_at->startOfSecond() : null;
                if (!$dbFile || $dbModifiedAtJst->lt($boxModifiedAtJst)) {
                    $filesToUpdate[] = $boxFile;
                }
            }
            
            if (empty($filesToUpdate)) return 0;

            $updateChunks = array_chunk($filesToUpdate, 200);
            foreach ($updateChunks as $chunkOfFiles) {
                $result = $this->fetchContentsWithRefresh($chunkOfFiles, $accessToken, $refreshToken, $urlPath);
                $downloadedContents = $result['contents'];
                $accessToken = $result['new_access_token'];
                $refreshToken = $result['new_refresh_token'];
                
                if (!empty($downloadedContents)) {
                    $dataToUpsert = [];
                    foreach ($downloadedContents as $fileId => $content) {
                        $boxFile = $boxFilesById->get($fileId);
                        if ($boxFile) {
                            $dataToUpsert[] = [ /* ... */ ];
                        }
                    }
                    if (!empty($dataToUpsert)) {
                        ModelFileCache::upsert($dataToUpsert, ['box_file_id'], [/* ... */]);
                    }
                }
            }
            $this->cleanupDeletedFiles($projectFolderId, $boxFilesById);
            return count($filesToUpdate);
        } catch (Throwable $e) { throw $e; }
    }
    
    private function fetchContentsWithRefresh(array $files, $accessToken, $refreshToken, $urlPath)
    {
        $maxRetries = 2;
        for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
            $result = $this->fetchMultipleBoxFileContentsConcurrently($files, $accessToken);
            if ($result['status'] === 'success') {
                return ['contents' => $result['contents'], 'new_access_token' => $accessToken, 'new_refresh_token' => $refreshToken];
            }
            if ($result['status'] === 'token_expired') {
                $newTokens = $this->refreshBoxToken($refreshToken, $urlPath);
                if (!$newTokens) throw new Exception("Failed to refresh Box token after expiration.");
                $accessToken = $newTokens['access_token'];
                $refreshToken = $newTokens['refresh_token'];
                continue;
            }
            throw new Exception($result['message'] ?? 'Unknown download error.');
        }
        throw new Exception("Failed to download files after {$maxRetries} refresh attempts.");
    }
    
    private function fetchMultipleBoxFileContentsConcurrently(array $files, $accessToken)
    {
        // ... (以前の回答と同じ、引数で$accessTokenを受け取るバージョン) ...
    }
    
    public function refreshBoxToken($refreshToken, $urlPath)
    {
        if (empty($refreshToken)) { Log::error("Refresh token is empty."); return null; }
        try {
            $clientId = env("BOX_CLIENT_ID"); $clientSecret = env("BOX_CLIENT_SECRET");
            if (strpos($urlPath, 'deployment') !== false) {
                $clientId = env("BOX_CLIENT_ID_FOR_DEV_SLOT"); $clientSecret = env("BOX_CLIENT_SECRET_FOR_DEV_SLOT");
            }
            if (!$clientId || !$clientSecret) { Log::error("Box credentials not set."); return null; }
            $response = Http::asForm()->post('https://api.box.com/oauth2/token', [ /* ... */ ]);
            if ($response->successful()) {
                $data = $response->json();
                $newAccessToken = $data['access_token'];
                $newRefreshToken = $data['refresh_token'];
                $this->saveBoxTokens($newAccessToken, $newRefreshToken); // DBを更新
                return ['access_token' => $newAccessToken, 'refresh_token' => $newRefreshToken];
            } else { /* ... (エラーハンドリング) ... */ }
        } catch (Exception $e) { /* ... */ }
    }
    
    public function saveBoxTokens($accessToken, $refreshToken)
    {
        DB::connection('dldwh')->table('box_tokens')->updateOrInsert(
            ['id' => 1],
            ['access_token' => $accessToken, 'refresh_token' => $refreshToken, 'login_time' => time(), 'updated_at' => now()]
        );
    }

    public function getLatestBoxTokens()
    {
        return DB::connection('dldwh')->table('box_tokens')->first();
    }
    
    // ... (job_logsテーブル操作のヘルパーメソッドは変更なし) ...
}```

#### 2. `app/Jobs/SyncBoxProject.php` （ジョブ）

`wnp`系のロジックを排除し、DBから取得したトークンをModelに渡す責務に専念します。

```php
<?php

namespace App\Jobs;

use App\Models\DLDHWDataImportModel;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;
use Throwable;

class SyncBoxProject implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $projectFolderId;
    protected $urlPath;

    public function __construct($projectFolderId, $urlPath)
    {
        $this->projectFolderId = $projectFolderId;
        $this->urlPath = $urlPath;
    }

    public function handle()
    {
        Log::info("Job started for project: {$this->projectFolderId}");
        $dldwhModel = new DLDHWDataImportModel();
        
        try {
            $tokens = $dldwhModel->getLatestBoxTokens();
            if (!$tokens) {
                throw new \Exception("No valid tokens found in the database. Please log in again.");
            }
            
            $filesUpdatedCount = $dldwhModel->syncAndGetModelData(
                $this->projectFolderId, 
                $tokens->access_token, 
                $tokens->refresh_token,
                $this->urlPath
            );

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


<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use App\Jobs\SyncBoxProject;
use Illuminate\Http\Request;

class DLDWHDataObjectViewerController extends Controller
{
    // ... (objViewer, getModelData, checkSyncJobStatus, getProjectList, getCategoryNameByElementId は変更なし) ...

    /**
     * 【最終版】同期ジョブを開始させる
     */
    public function startSync(Request $request)
    {
        if (!session()->has('access_token')) {
            return response()->json(['status' => 'no_token']);
        }

        $projectFolderId = $request->input('folderId');
        if (empty($projectFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }
        
        $dldwhModel = new DLDHWDataImportModel();
        
        // 【重要】DBから最新のトークンを取得し、セッションを同期する
        $latestTokens = $dldwhModel->getLatestBoxTokens();
        if ($latestTokens) {
            session([
                'access_token' => $latestTokens->access_token,
                'refresh_token' => $latestTokens->refresh_token,
                'box_login_time' => $latestTokens->login_time,
            ]);
            Log::info("Session updated with latest tokens from database before dispatching job.");
        }

        $dldwhModel->deleteFinishedJobLog($projectFolderId);
        $dldwhModel->deleteFailedJobs();
        
        SyncBoxProject::dispatch($projectFolderId, url('/'));
        
        return response()->json(['status' => 'job_dispatched']);
    }
    
    /**
     * Box認証後にDBとセッションにトークンを保存する
     */
    public function saveAccessTokenAfterLogin($accessToken, $refreshToken)
    {
        session(['access_token' => $accessToken, 'refresh_token' => $refreshToken, 'box_login_time' => time()]);
        $dldwhModel = new DLDHWDataImportModel();
        $dldwhModel->saveBoxTokens($accessToken, $refreshToken);
    }
}
