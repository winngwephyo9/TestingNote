<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use App\Jobs\SyncBoxProject;
use Illuminate\Http\Request;
use GuzzleHttp\Client;
use Illuminate\Support\Facades\Log;
use Exception;

class DLDWHDataObjectViewerController extends Controller
{
    public function objViewer() { return view('DLDWH.OBJViewer'); }

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
        $dldwhModel->deleteFinishedJobLog($projectFolderId);
        $dldwhModel->deleteFailedJobs();
        SyncBoxProject::dispatch($projectFolderId, url('/'));
        return response()->json(['status' => 'job_dispatched']);
    }

    public function checkSyncJobStatus(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        $dldwhModel = new DLDHWDataImportModel();
        $jobLog = $dldwhModel->getFinishedJobLog($projectFolderId);
        if ($jobLog) {
            return response()->json(['status' => $jobLog->status, 'message' => $jobLog->message]);
        }
        $failedJob = $dldwhModel->getFailedJob($projectFolderId);
        if ($failedJob) {
            return response()->json(['status' => 'failed', 'message' => 'A job has failed unexpectedly. Check system logs.']);
        }
        return response()->json(['status' => 'processing']);
    }

    public function getModelData(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        $dldwhModel = new DLDHWDataImportModel();
        $modelData = $dldwhModel->getCachedModelData($projectFolderId);
        if (isset($modelData['error'])) {
            return response()->json($modelData, 404);
        }
        return response()->json($modelData);
    }
    
    public function getProjectList(Request $request)
    {
        $mainFolderId = $request->input('folderId', '339110566808');
        $dldwhModel = new DLDHWDataImportModel();
        if (session()->has('access_token')) {
            try {
                $client = new Client(['verify' => false]);
                $requestURL = "https://api.box.com/2.0/folders/" . $mainFolderId . "/items?fields=id,name";
                $header = ["Authorization" => "Bearer " . session('access_token')];
                $response = $client->request('GET', $requestURL, ['headers' => $header]);
                $items = json_decode($response->getBody()->getContents())->entries;
                $projects = [];
                foreach ($items as $item) {
                    if ($item->type == "folder") {
                        $projects[] = ['id' => $item->id, 'name' => $item->name];
                    }
                }
                usort($projects, fn($a, $b) => strcmp($a['name'], $b['name']));
                return response()->json(['login_status' => 'logged_in', 'projects' => $projects]);
            } catch (Exception $e) {
                $cachedProjects = $dldwhModel->getCachedProjectList();
                return response()->json(['login_status' => 'error', 'projects' => $cachedProjects]);
            }
        } else {
            $cachedProjects = $dldwhModel->getCachedProjectList();
            return response()->json(['login_status' => 'logged_out', 'projects' => $cachedProjects]);
        }
    }
    
    public function getCategoryNameByElementId(Request $request)
    {
        $WSCenID = $request->input('WSCenID');
        $elementIds = $request->input('ElementIds');
        $dldwhModel = new DLDHWDataImportModel();
        return $dldwhModel->getCategoryNameByElementId($WSCenID, $elementIds);
    }
    
    public function saveAccessTokenAfterLogin($accessToken, $refreshToken)
    {
        session(['access_token' => $accessToken, 'refresh_token' => $refreshToken, 'box_login_time' => time()]);
        $dldwhModel = new DLDHWDataImportModel();
        $dldwhModel->saveBoxTokens($accessToken, $refreshToken);
    }
}```

---

### Part 3: `app/Jobs/SyncBoxProject.php` （ジョブ）

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

    public $timeout = 0;
    public $tries = 1;
    public $failOnTimeout = true;

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

namespace App\Models;

use App\Models\ModelFileCache;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Http;
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Exception\ClientException;
use GuzzleHttp\Psr7\Request as GuzzleRequest;
use GuzzleHttp\RequestOptions;
use Illuminate\Support\Carbon;
use Exception;
use Throwable;

class DLDHWDataImportModel extends Model
{
    // ... getCategoryNameByElementId, getCachedProjectList は変更なし ...

    public function getCachedModelData($projectFolderId)
    {
        $cachedFiles = ModelFileCache::where('project_box_id', $projectFolderId)->get()->groupBy('base_name');
        if ($cachedFiles->isEmpty()) return ['error' => 'No cached model found.'];

        $objMtlPairs = [];
        foreach ($cachedFiles as $baseName => $files) {
            $objFile = $files->firstWhere('file_type', 'obj');
            if ($objFile && $objFile->file_path && Storage::disk('local')->exists($objFile->file_path)) {
                try {
                    $objContent = Storage::disk('local')->get($objFile->file_path);
                    $mtlContent = null;
                    $mtlFile = $files->firstWhere('file_type', 'mtl');
                    if ($mtlFile && $mtlFile->file_path && Storage::disk('local')->exists($mtlFile->file_path)) {
                        $mtlContent = Storage::disk('local')->get($mtlFile->file_path);
                    }
                    $objMtlPairs[] = [
                        'baseName' => $baseName,
                        'obj' => ['name' => $objFile->file_name, 'content' => $objContent],
                        'mtl' => $mtlFile ? ['name' => $mtlFile->file_name, 'content' => $mtlContent] : null
                    ];
                } catch(Exception $e) {
                    Log::error("Could not read file from storage: " . $e->getMessage());
                }
            }
        }
        return $objMtlPairs;
    }

    public function syncAndGetModelData($projectFolderId, $accessToken, $refreshToken, $urlPath)
    {
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
                $result = $this->fetchAndStreamBoxFilesConcurrently($chunkOfFiles, $projectFolderId, $projectName, $accessToken, $refreshToken, $urlPath);
                $downloadedFilesMetadata = $result['metadata'];
                $accessToken = $result['new_access_token'];
                $refreshToken = $result['new_refresh_token'];
                if (!empty($downloadedFilesMetadata)) {
                    ModelFileCache::upsert($downloadedFilesMetadata, ['box_file_id'], ['project_name', 'file_name', 'base_name', 'file_type', 'file_path', 'box_modified_at', 'updated_at']);
                }
            }
            $this->cleanupDeletedFiles($projectFolderId, $boxFilesById);
            return count($filesToUpdate);
        } catch (Throwable $e) { throw $e; }
    }
    
    private function fetchAndStreamBoxFilesConcurrently($files, $projectFolderId, $projectName, $accessToken, $refreshToken, $urlPath)
    {
        $maxRetries = 2;
        for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
            $result = $this->streamFilesWithGuzzle($files, $projectFolderId, $projectName, $accessToken);
            if ($result['status'] === 'success') {
                return ['metadata' => $result['metadata'], 'new_access_token' => $accessToken, 'new_refresh_token' => $refreshToken];
            }
            if ($result['status'] === 'token_expired') {
                $newTokens = $this->refreshBoxToken($refreshToken, $urlPath);
                if (!$newTokens) throw new Exception("Failed to refresh token.");
                $accessToken = $newTokens['access_token'];
                $refreshToken = $newTokens['refresh_token'];
                continue;
            }
            throw new Exception("Download failed.");
        }
        throw new Exception("Failed after multiple refresh attempts.");
    }
    
    private function streamFilesWithGuzzle($files, $projectFolderId, $projectName, $accessToken, &$downloadedFilesMetadata = [])
    {
        $tokenExpired = false;
        $client = new Client(['verify' => false]);
        $header = ["Authorization" => "Bearer " . $accessToken];
        $storagePath = Storage::disk('local')->getAdapter()->getPathPrefix();
        $directory = "{$storagePath}model_files/{$projectFolderId}/";
        if (!file_exists($directory)) mkdir($directory, 0775, true);

        $requests = function ($files) use ($header, $directory) {
            foreach ($files as $file) {
                $url = "https://api.box.com/2.0/files/{$file['id']}/content";
                $sinkPath = "{$directory}{$file['id']}_{$file['name']}";
                yield $file['id'] => new GuzzleRequest('GET', $url, $header, null, ['sink' => $sinkPath]);
            }
        };
        $pool = new Pool($client, $requests($files), [
            'concurrency' => 10,
            'fulfilled' => function ($response, $fileId) use (&$downloadedFilesMetadata, $projectFolderId, $projectName, $files) {
                $boxFile = collect($files)->firstWhere('id', $fileId);
                if ($boxFile) {
                    $filePath = "model_files/{$projectFolderId}/{$fileId}_{$boxFile['name']}";
                    $downloadedFilesMetadata[] = [
                        'project_box_id' => $projectFolderId, 'project_name' => $projectName,
                        'file_name' => $boxFile['name'], 'base_name' => pathinfo($boxFile['name'], PATHINFO_FILENAME),
                        'box_file_id' => $fileId, 'file_type' => strtolower(pathinfo($boxFile['name'], PATHINFO_EXTENSION)),
                        'file_path' => $filePath, 'box_modified_at' => Carbon::parse($boxFile['modified_at'])->setTimezone('Asia/Tokyo'),
                        'created_at' => now(), 'updated_at' => now(),
                    ];
                }
            },
            'rejected' => function ($reason, $fileId) use (&$tokenExpired) {
                if ($reason instanceof ClientException && $reason->getResponse()->getStatusCode() == 401) {
                    $tokenExpired = true;
                } else { Log::error("Failed to stream file ID {$fileId}: " . $reason->getMessage()); }
            }
        ]);
        $pool->promise()->wait();
        if ($tokenExpired) return ['status' => 'token_expired'];
        return ['status' => 'success', 'metadata' => $downloadedFilesMetadata];
    }
    
    // ... (fetchProjectName, fetchFullBoxFileList, cleanupDeletedFiles, refreshBoxToken, DBヘルパーメソッドなど、
    // 前回の回答で完成したものをここに含めます) ...
}
