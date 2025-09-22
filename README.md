<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use GuzzleHttp\Client;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Exception;
use App\Jobs\SyncBoxProject;
use Illuminate\Support\Facades\Cache;

class DLDWHDataObjectViewerController extends Controller
{
    protected $access_token;
    protected $refresh_token;
    protected $box_login_time;
    protected $url_path;

    protected $projectFolderId;

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
        // if (!session()->has('access_token') || empty(session('access_token'))) {
        //     return response()->json(['status' => 'no_token']);
        // }
        $finishJob = [];
        if (session()->has('access_token')) {
            $this->access_token = session('access_token');
            if ($this->access_token == "") {
                $finishJob = [
                    'token' => "no_token",
                ];
                return response()->json($finishJob);
            }

            if (session()->has('box_login_time')) {
                $this->box_login_time = session('box_login_time');
                Log::info('box_login_time In session ' . $this->box_login_time);
            }
            if (session()->has('refresh_token')) {
                $this->refresh_token = session('refresh_token');
                Log::info('refresh_token in Session' . $this->refresh_token);
            }
            if (session()->has('url_path')) {
                $this->url_path = session('url_path');
                Log::info('url_path in Session ' . $this->url_path);
            }

            $this->projectFolderId = $request->input('folderId');
            if (empty($this->projectFolderId)) {
                return response()->json(['error' => 'Folder ID is required.'], 400);
            }

            $dldwhModel = new DLDHWDataImportModel();

            // 以前のジョブログと失敗ジョブを削除
            $dldwhModel->deleteFinishedJobLog($this->projectFolderId);
            $dldwhModel->deleteFailedJobs();
            // $dldwhModel = new DLDHWDataImportModel();
            $tokens = $dldwhModel->getLatestBoxTokens();
            if ($tokens) {
                $this->access_token = $tokens->access_token;
                $this->refresh_token = $tokens->refresh_token;
                $this->box_login_time = $tokens->login_time;

                // throw new \Exception("No valid tokens found in the database. Please log in again.");
            }

            // $accessToken = $tokens->access_token;
            // $refreshToken = $tokens->refresh_token;
            // $loginTime = $tokens->login_time;

            // $this->saveAccessTokenAfterLogin(session('access_token'), session('refresh_token'));
            // SyncBoxProject::dispatch($projectFolderId, session('url_path'));
            Log::info('refresh_token before Job' . $this->refresh_token);

            SyncBoxProject::dispatch(
                $this->projectFolderId,
                $this->access_token,
                $this->refresh_token,
                $this->box_login_time,
                $this->url_path
            );
            $finishJob = [
                'isfinishJob' => false,
            ];
            return response()->json(['status' => 'job_dispatched']);
        } else {
            $finishJob = [
                'token' => "no_token",
            ];
            return response()->json($finishJob);
        }
        // return response()->json(['status' => 'job_dispatched']);
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
    // public function saveAccessTokenAfterLogin($accessToken, $refreshToken)
    // {
    //     // セッションに保存
    //     session([
    //         'access_token' => $accessToken,
    //         'refresh_token' => $refreshToken,
    //         'box_login_time' => time(),
    //         'url_path' => url('/')
    //     ]);

    //     // DBにも保存
    //     $dldwhModel = new DLDHWDataImportModel();
    //     $dldwhModel->saveBoxTokens($accessToken, $refreshToken);
    // }
}
