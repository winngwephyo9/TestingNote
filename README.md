<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use App\Jobs\SyncBoxProject;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Cache;
use GuzzleHttp\Client;
use Exception;

class DLDWHDataObjectViewerController extends Controller
{
    /**
     * OBJビューアのメインビューを表示する
     */
    public function objViewer()
    {
        return view('DLDWH.OBJViewer');
    }

    /**
     * モデルデータを取得し、必要であれば同期ジョブを開始するメインAPI
     */
    public function getModelData(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        if (empty($projectFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }

        $jobStatusKey = "sync-status-{$projectFolderId}";
        $currentStatus = Cache::get($jobStatusKey);

        // Boxにログインしているかチェック
        if (session()->has('access_token') && !empty(session('access_token'))) {
            // 現在処理中でもなく、かつ完了直後でもない場合にのみディスパッチ
            if ($currentStatus !== 'processing' && $currentStatus !== 'completed') {
                Log::info("Dispatching SyncBoxProject job for project: {$projectFolderId}. Previous status was: " . ($currentStatus ?? 'null'));
                
                // ジョブ投入前に「実行中」の状態をキャッシュに保存
                Cache::put($jobStatusKey, 'processing', now()->addHours(12));
                
                // ジョブに渡すセッションデータを準備
                $sessionData = [
                    'access_token' => session('access_token'),
                    'refresh_token' => session('refresh_token'),
                    'box_login_time' => session('box_login_time', time()),
                    'url_path' => url('/') // ジョブに現在のURLパスを渡す
                ];

                // ジョブをキューに投入する
                SyncBoxProject::dispatch($projectFolderId, $sessionData);
            } else {
                 Log::info("Sync job for project {$projectFolderId} is already {$currentStatus}. Skipping dispatch.");
            }
        }
        
        $dldwhModel = new DLDHWDataImportModel();
        $modelData = $dldwhModel->getCachedModelData($projectFolderId);

        // レスポンス構造を統一
        $responseData = [
            'sync_status' => Cache::get($jobStatusKey, 'completed'),
            'objMtlPairs' => [],
        ];

        if (isset($modelData['error'])) {
            $responseData['error'] = $modelData['error'];
        } else {
            $responseData['objMtlPairs'] = $modelData;
        }
        
        return response()->json($responseData);
    }

    /**
     * フロントエンドが同期状態を問い合わせるためのAPI
     */
    public function getSyncStatus(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        if (empty($projectFolderId)) {
            return response()->json(['sync_status' => 'error', 'message' => 'Folder ID is required.'], 400);
        }
        $jobStatusKey = "sync-status-{$projectFolderId}";
        $status = Cache::get($jobStatusKey, 'completed');
        return response()->json(['sync_status' => $status]);
    }
    
    /**
     * ログイン状態に応じてBoxまたはDBキャッシュからプロジェクトリストを返す
     */
    public function getProjectList(Request $request)
    {
        $mainFolderId = $request->input('folderId', '339110566808'); // デフォルトIDを設定
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
     * 要素の詳細情報を取得する
     */
    public function getCategoryNameByElementId(Request $request)
    {
        $WSCenID = $request->input('WSCenID');
        $elementIds = $request->input('ElementIds');
        $dldwhModle = new DLDHWDataImportModel();
        return $dldwhModle->getCategoryNameByElementId($WSCenID, $elementIds);
    }
    
    /**
     * Box認証後などに呼び出され、トークンとログイン時刻をセッションに保存する
     */
    public function saveAccessTokenAfterLogin($accessToken, $refreshToken)
    {
        session([
            'access_token' => $accessToken,
            'refresh_token' => $refreshToken,
            'box_login_time' => time(),
            'url_path' => url('/')
        ]);
    }
}
