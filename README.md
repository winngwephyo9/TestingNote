In the controller
class DLDWHDataObjectViewerController extends Controller
{
    /**
     * Display the 全体物件 Object Viewer page.
     *
     * @return \Illuminate\View\View The view for the Bara Object Viewer.
     */
    public function objViewer()
    {
        return view('DLDWH.OBJViewer');
    }

    /* Retrieve category names based on element IDs.
     *
     * @param \Illuminate\Http\Request $request The incoming HTTP request containing 'WSCenID' and 'ElementIds'.
     * @return mixed The result from the model's `getCategoryNameByElementId` method, typically a list of category names.
    */
    public function getCategoryNameByElementId(Request $request)
    {
        $WSCenID = $request->input('WSCenID');
        $elementIds = $request->input('ElementIds');
        $dldwhModle = new DLDHWDataImportModel();
        return $dldwhModle->getCategoryNameByElementId($WSCenID, $elementIds);
    }

    /**
     * Display the Bara Object Viewer page.
     *
     * @return \Illuminate\View\View The view for the Bara Object Viewer.
     */

    public function bara_objViewer()
    {
        return view("DLDWH.Bara_OBJViewer");
    }

    /**
     * 【リファクタリング版】同期ジョブを開始させる
     */
    public function startSync(Request $request)
    {
        if (!session()->has('access_token') || empty(session('access_token'))) {
            return response()->json(['status' => 'no_token']);
        }

        $projectFolderId = $request->input('folderId');
        if (empty($projectFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }

        $dldwhModel = new DLDHWDataImportModel();

        // 以前のジョブログと失敗ジョブを削除
        $dldwhModel->deleteFinishedJobLog($projectFolderId);
        $dldwhModel->deleteFailedJobs();

        $sessionData = [
            'access_token' => session('access_token'),
            'refresh_token' => session('refresh_token'),
            'box_login_time' => session('box_login_time', time()),
            'url_path' => url('/')
        ];
        $this->saveAccessTokenAfterLogin(session('access_token'), session('refresh_token'));
        SyncBoxProject::dispatch($projectFolderId, $sessionData);

        return response()->json(['status' => 'job_dispatched']);
    }

    /**
     * 【リファクタリング版】ジョブの状態を確認する
     */
    public function checkSyncJobStatus(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        $dldwhModel = new DLDHWDataImportModel();

        $jobLog = $dldwhModel->getFinishedJobLog($projectFolderId);
        if ($jobLog) {
            // job_logsテーブルに記録があれば、それを返す
            return response()->json(['status' => $jobLog->status, 'message' => $jobLog->message]);
        }

        // Laravelの標準failed_jobsテーブルも念のためチェック
        $failedJob = $dldwhModel->getFailedJobs();
        if ($failedJob) {
            // payloadを解析して、どのプロジェクトか特定するのが望ましいが、ここでは簡易的に
            return response()->json(['status' => 'failed', 'message' => 'A job has failed. Check logs.']);
        }

        // どちらのテーブルにも記録がなければ、処理中とみなす
        return response()->json(['status' => 'processing']);
    }

    /**
     * キャッシュされたモデルデータを取得する
     */
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

    /**
     * ログイン状態に応じてBoxまたはDBキャッシュからプロジェクトリストを返す
     */
    public function getProjectList(Request $request)
    {
        $mainFolderId = $request->input('folderId');
        $dldwhModel = new DLDHWDataImportModel();

        if (session()->has('access_token') && !empty(session('access_token'))) {
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

                usort($projects, function ($a, $b) {
                    return strcmp($a['name'], $b['name']);
                });

                return response()->json(['login_status' => 'logged_in', 'projects' => $projects]);
            } catch (Exception $e) {
                Log::error("Failed to read project list from Box: " . $e->getMessage());
                $cachedProjects = $dldwhModel->getCachedProjectList();
                return response()->json(['login_status' => 'error', 'projects' => $cachedProjects]);
            }
        } else {
            $cachedProjects = $dldwhModel->getCachedProjectList();
            return response()->json(['login_status' => 'logged_out', 'projects' => $cachedProjects]);
        }
    }

    /**
     * Box認証後などに呼び出され、トークンとログイン時刻をセッションに保存する
     */
    public function saveAccessTokenAfterLogin($accessToken, $refreshToken)
    {
        // セッションに保存
        session([
            'access_token' => $accessToken,
            'refresh_token' => $refreshToken,
            'box_login_time' => time(),
            'url_path' => url('/')
        ]);

        // DBにも保存
        $dldwhModel = new DLDHWDataImportModel();
        $dldwhModel->saveBoxTokens($accessToken, $refreshToken);
    }
}


IN the job
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
        Log::info("Job started for project: {$this->projectFolderId}");

        try {
            // 【重要】ジョブ開始時にDBから最新のトークン情報を取得
            $dldwhModel = new DLDHWDataImportModel();
            $latestTokens = $dldwhModel->getLatestBoxTokens();
            if (!$latestTokens) {
                session($this->sessionData);
                // throw new Exception("No valid tokens found in the database.");
            }

            // 取得した最新情報をセッションにセットして、後続の処理で使えるようにする
            session([
                'access_token' => $latestTokens->access_token,
                'refresh_token' => $latestTokens->refresh_token,
                'box_login_time' => $latestTokens->login_time,
                'url_path' => url('/') // url_pathはここで設定
            ]);

            $dldwhModel->checkBoxExpiryStatus(); // トークンチェックを実行
            // syncAndGetModelDataは更新されたファイル数を返すように修正済みと仮定
            $filesUpdatedCount = $dldwhModel->syncAndGetModelData($this->projectFolderId);

            $status = ($filesUpdatedCount > 0) ? 'completed' : 'no_files_to_update';
            $this->saveJobLog($status);

            Log::info("Successfully completed sync job for project: {$this->projectFolderId} with status: {$status}");
        } catch (Throwable $e) {
            $this->fail($e); // 失敗した場合はfailed()メソッドを呼び出す
        }
    }

    /**
     * job_logsテーブルに結果を保存する
     */
    private function saveJobLog($status, $message = null)
    {
        $dldwhModel = new DLDHWDataImportModel();
        // 【修正済み】余分な$を削除
        $dldwhModel->jobLogsUpdateOrInsert($this->projectFolderId, $status, $message);
    }


    /**
     * ジョブ内でトークンの有効期限をチェックし、更新するメソッド
     */
    private function checkBoxExpiryStatus()
    {
        $loginTime = session('box_login_time', 0);
        if (time() > $loginTime + (60 * 50)) {
            Log::info('Box token is about to expire. Refreshing...');
            $dldwhModel = new DLDHWDataImportModel();
            // refreshBoxTokenは中でDBとセッションを両方更新してくれる
            $newTokens = $dldwhModel->refreshBoxToken(session('refresh_token'), session('url_path'));
            if (!$newTokens) {
                throw new Exception("Failed to refresh Box token.");
            }
        }
    }

    /**
     * ジョブが最終的に失敗したときに呼び出される
     */
    public function failed(Throwable $exception)
    {
        $this->saveJobLog('failed', $exception->getMessage());
        Log::error("Sync job has failed for project {$this->projectFolderId}: " . $exception->getMessage());
    }
}

In the model


    /**
     * 【修正版】DBキャッシュから、保存されているプロジェクトのリストを取得する
     */
    public function getCachedProjectList()
    {
        $projects = ModelFileCache::select('project_box_id', 'project_name')
            ->whereNotNull('project_name')->distinct()->get()
            ->map(function ($file) {
                return ['id' => $file->project_box_id, 'name' => $file->project_name];
            });
        return $projects->sortBy('name')->values();
    }

    /**
     * DBキャッシュからモデルデータを取得する
     */
    public function getCachedModelData($projectFolderId)
    {
        $cachedFiles = ModelFileCache::where('project_box_id', $projectFolderId)->get()->groupBy('base_name');
        if ($cachedFiles->isEmpty()) {
            return ['error' => 'No cached model found. Please log in to Box to sync data.'];
        }
        return $this->formatDataForFrontend($cachedFiles);
    }


    /**
     * 【修正版】同期時にプロジェクト名をDBに保存する
     */
    public function syncAndGetModelData($projectFolderId)
    {
        mb_internal_encoding('UTF-8');
        set_time_limit(0);

        try {
            $accessToken = session('access_token');
            if (empty($accessToken)) throw new Exception("Box access token is missing.");

            $client = new Client(['verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken];
            $folderInfoUrl = "https://api.box.com/2.0/folders/{$projectFolderId}?fields=name";
            $folderInfoResponse = $client->get($folderInfoUrl, ['headers' => $header]);
            $projectName = json_decode($folderInfoResponse->getBody()->getContents())->name;

            $boxFiles = $this->fetchFullBoxFileList($projectFolderId, $accessToken, $client);
            $boxFilesById = collect($boxFiles)->keyBy('id');
            $dbFiles = ModelFileCache::where('project_box_id', $projectFolderId)->get()->keyBy('box_file_id');

            $filesToUpdate = [];
            foreach ($boxFiles as $boxFile) {
                $dbFile = $dbFiles->get($boxFile['id']);
                if (!$dbFile) {
                    $filesToUpdate[] = $boxFile;
                    continue;
                }
                $boxModifiedAtJst = Carbon::parse($boxFile['modified_at'])->setTimezone('Asia/Tokyo')->startOfSecond();
                $dbModifiedAtJst = $dbFile->box_modified_at->startOfSecond();
                if ($dbModifiedAtJst->lt($boxModifiedAtJst)) {
                    $filesToUpdate[] = $boxFile;
                }
            }

            if (!empty($filesToUpdate)) {
                $updateChunks = array_chunk($filesToUpdate, 1000);
                foreach ($updateChunks as $chunkIndex => $chunkOfFiles) {
                    $this->checkBoxExpiryStatus();
                    Log::info("Processing chunk " . ($chunkIndex + 1) . "/" . count($updateChunks) . " (" . count($chunkOfFiles) . " files)...");
                    $result = $this->fetchMultipleBoxFileContentsConcurrently($chunkOfFiles, $client);
                    if ($result['status'] === 'error') throw new Exception($result['message']);

                    $downloadedContents = $result['contents'];
                    $dataToUpsert = [];
                    foreach ($downloadedContents as $fileId => $content) {
                        $boxFile = $boxFilesById->get($fileId);
                        if ($boxFile) {
                            $dataToUpsert[] = [
                                'project_box_id' => $projectFolderId,
                                'project_name' => $projectName,
                                'file_name' => $boxFile['name'],
                                'base_name' => pathinfo($boxFile['name'], PATHINFO_FILENAME),
                                'box_file_id' => $fileId,
                                'file_type' => strtolower(pathinfo($boxFile['name'], PATHINFO_EXTENSION)),
                                'content' => $content,
                                'box_modified_at' => Carbon::parse($boxFile['modified_at'])->setTimezone('Asia/Tokyo'),
                                'created_at' => now(),
                                'updated_at' => now(),
                            ];
                        }
                    }
                    if (!empty($dataToUpsert)) {
                        ModelFileCache::upsert($dataToUpsert, ['box_file_id'], ['project_box_id', 'project_name', 'file_name', 'base_name', 'file_type', 'content', 'box_modified_at', 'updated_at']);
                    }
                }
            }

            $boxFileIds = $boxFilesById->keys()->all();
            $dbFileIds = $dbFiles->keys()->all();
            $deletedIds = array_diff($dbFileIds, $boxFileIds);
            if (!empty($deletedIds)) {
                ModelFileCache::whereIn('box_file_id', $deletedIds)->delete();
            }
        } catch (Throwable $e) {
            Log::error("Box Sync Failed: " . $e->getMessage(), ['exception' => $e]);
            throw $e;
        }
        return true;
    }

    /**
     * フロントエンド用にデータを整形する
     */
    private function formatDataForFrontend($groupedFiles)
    {
        $objMtlPairs = [];
        foreach ($groupedFiles as $baseName => $files) {
            $objFile = $files->firstWhere('file_type', 'obj');
            $mtlFile = $files->firstWhere('file_type', 'mtl');
            if ($objFile) {
                $objMtlPairs[] = [
                    'baseName' => $baseName,
                    'obj' => ['name' => $objFile->file_name, 'content' => $objFile->content],
                    'mtl' => $mtlFile ? ['name' => $mtlFile->file_name, 'content' => $mtlFile->content] : null
                ];
            }
        }
        return $objMtlPairs;
    }

    /**
     * Boxからファイルリストを全て取得（更新日時も含む）
     */
    private function fetchFullBoxFileList($projectFolderId, $accessToken, $client)
    {
        $header = ["Authorization" => "Bearer " . $accessToken];
        $allFolderItems = [];
        $offset = 0;
        $limit = 1000;
        do {
            $requestURL = "https://api.box.com/2.0/folders/{$projectFolderId}/items?fields=id,name,modified_at&limit={$limit}&offset={$offset}";
            $response = $client->get($requestURL, ['headers' => $header]);
            $body = json_decode($response->getBody()->getContents(), true);
            if (isset($body['entries'])) {
                $allFolderItems = array_merge($allFolderItems, $body['entries']);
                $offset += count($body['entries']);
            } else {
                break;
            }
        } while (isset($body['total_count']) && $offset < $body['total_count']);
        return $allFolderItems;
    }

    /**
     * トークン切れに対応した堅牢な並列ダウンロード関数
     */
    private function fetchMultipleBoxFileContentsConcurrently($files, $client)
    {
        $downloadedContents = [];
        $tokenExpired = false;
        $header = ["Authorization" => "Bearer " . session('access_token')];
        $requests = function ($files) use ($header) {
            foreach ($files as $file) {
                $url = "https://api.box.com/2.0/files/{$file['id']}/content";
                yield $file['id'] => new \GuzzleHttp\Psr7\Request('GET', $url, $header);
            }
        };
        $pool = new Pool($client, $requests($files), [
            'concurrency' => 10,
            'fulfilled' => function ($response, $fileId) use (&$downloadedContents) {
                $downloadedContents[$fileId] = $response->getBody()->getContents();
            },
            'rejected' => function ($reason, $fileId) {
                // Log::error("Failed to download file ID {$fileId}: " . $reason->getMessage());
            }
        ]);
        $promise = $pool->promise();
        $promise->wait();
        return ['status' => 'success', 'contents' => $downloadedContents];
    }

    /**
     * 【新規】DBから最新のBoxトークン情報を取得する
     */
    public function getLatestBoxTokens()
    {
        return DB::connection('dldwh')->table('box_tokens')->first();
    }

    /**
     * 【新規】DBに新しいBoxトークン情報を保存/更新する
     */
    public function saveBoxTokens($accessToken, $refreshToken)
    {
        DB::connection('dldwh')->table('box_tokens')->updateOrInsert(
            ['id' => 1], // 常にID=1の行を更新
            [
                'access_token' => $accessToken,
                'refresh_token' => $refreshToken,
                'login_time' => time(),
                'updated_at' => now()
            ]
        );
    }

    public function refreshBoxToken($refreshToken, $urlPath)
    {
        if (empty($refreshToken)) {
            Log::error("Refresh token is empty.");
            return null;
        }
        try {
            $clientId = env("BOX_CLIENT_ID");
            $clientSecret = env("BOX_CLIENT_SECRET");
            if (strpos($urlPath, 'deployment') !== false) {
                $clientId = env("BOX_CLIENT_ID_FOR_DEV_SLOT");
                $clientSecret = env("BOX_CLIENT_SECRET_FOR_DEV_SLOT");
            }
            if (!$clientId || !$clientSecret) {
                Log::error("Box client credentials are not set.");
                return null;
            }
            $response = Http::asForm()->post('https://api.box.com/oauth2/token', [
                'grant_type' => 'refresh_token',
                'refresh_token' => $refreshToken,
                'client_id' => $clientId,
                'client_secret' => $clientSecret,
            ]);
            if ($response->successful()) {
                $data = $response->json();
                $newAccessToken = $data['access_token'];
                $newRefreshToken = $data['refresh_token'];

                // 【重要】DBとセッションの両方を更新
                $this->saveBoxTokens($newAccessToken, $newRefreshToken);
                session(['access_token' => $newAccessToken, 'refresh_token' => $newRefreshToken, 'box_login_time' => time()]);

                Log::info("Successfully refreshed and saved Box tokens.");
                return ['access_token' => $newAccessToken, 'refresh_token' => $newRefreshToken];
            } else {
                Log::error("Failed to refresh Box token.", ['status' => $response->status(), 'body' => $response->body()]);
                return null;
            }
        } catch (Exception $e) {
            Log::error("Exception on refresh token: " . $e->getMessage());
            return null;
        }
    }

    /**
     * saveBoxExpiryStatus
     * トーケンを更新した後、記録する
     *
     * @return void
     */
    // public function saveBoxExpiryStatus()
    // {
    //     $query = "INSERT INTO box_expiry_status (status) VALUES (?)";
    //     $result = DB::connection('dldwh')->insert($query, ['true']);
    // }

    /**
     * ジョブ内でトークンの有効期限をチェックし、更新するメソッド
     */
    public function checkBoxExpiryStatus()
    {
        $loginTime = session('box_login_time', 0);
        if (time() > $loginTime + (60 * 50)) {
            Log::info('Box token is about to expire. Refreshing...');
            // refreshBoxTokenは中でDBとセッションを両方更新してくれる
            $newTokens = $this->refreshBoxToken(session('refresh_token'), session('url_path'));
            if (!$newTokens) {
                throw new Exception("Failed to refresh Box token.");
            }
        }
    }

    /**
     * プロジェクトIDをキーとして、完了または失敗したジョブのログを取得する
     */
    public function getFinishedJobLog($projectId)
    {
        return DB::connection('dldwh')->table('job_logs_bara')->where('job_identifier', $projectId)->first();
    }

    /**
     * プロジェクトIDを指定して、古いジョブログを削除する
     */
    public function deleteFinishedJobLog($projectId)
    {
        DB::connection('dldwh')->table('job_logs_bara')->where('job_identifier', $projectId)->delete();
    }

    /**
     * Summary of jobLogsUpdateOrInsert
     * @param mixed $projectFolderId
     * @param mixed $status
     * @param mixed $message
     * @return void
     */
    public function jobLogsUpdateOrInsert($projectFolderId, $status, $message)
    {
        DB::connection('dldwh')->table('job_logs_bara')->updateOrInsert(
            ['job_identifier' => $projectFolderId],
            ['status' => $status, 'message' => $message, 'updated_at' => now()]
        );
    }
}


<img width="1220" height="885" alt="image" src="https://github.com/user-attachments/assets/41796c29-a4b0-40ca-9a1e-a69290766ad9" />
