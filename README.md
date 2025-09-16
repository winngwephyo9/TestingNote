<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use App\Jobs\SyncBoxProject;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Cache;

class DLDWHDataObjectViewerController extends Controller
{
    /**
     * 【修正版】返すJSONの構造を常に統一する
     */
    public function getModelData(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        if (empty($projectFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }

        $jobStatusKey = "sync-status-{$projectFolderId}";

        if (session()->has('access_token') && !empty(session('access_token'))) {
            if (!Cache::has($jobStatusKey)) {
                Log::info("Dispatching SyncBoxProject job for project: {$projectFolderId}");
                Cache::put($jobStatusKey, 'processing', now()->addHours(12));
                SyncBoxProject::dispatch(
                    $projectFolderId,
                    session('access_token'),
                    session('refresh_token')
                );
            }
        }
        
        $dldwhModel = new DLDHWDataImportModel();
        // モデルからデータを取得（エラーまたはペアの配列が返る）
        $modelData = $dldwhModel->getCachedModelData($projectFolderId);

        // =================================================================
        //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
        //
        //  **【重要】レスポンス構造の統一**
        //
        $responseData = [
            'sync_status' => Cache::get($jobStatusKey, 'completed'),
            'objMtlPairs' => [], // デフォルトは空の配列
        ];

        if (isset($modelData['error'])) {
            // エラーがある場合はエラーメッセージをセット
            $responseData['error'] = $modelData['error'];
        } else {
            // 正常な場合はファイルペアのデータをセット
            $responseData['objMtlPairs'] = $modelData;
        }
        
        return response()->json($responseData);
        //
        //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
        // =================================================================
    }

    // ... getSyncStatusは変更なし ...
}
