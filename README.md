<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use App\Jobs\SyncBoxProject;
use Illuminate\Http\Request;

class DLDWHDataObjectViewerController extends Controller
{
    public function objViewer() { return view('DLDWH.OBJViewer'); }

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
    
    // ... getProjectList, getCategoryNameByElementId, saveAccessTokenAfterLogin は変更なし ...
}
