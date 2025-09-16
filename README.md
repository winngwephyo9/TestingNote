**注意:** このロジックを機能させるには、`.env`ファイルにBoxアプリの`BOX_CLIENT_ID`と`BOX_CLIENT_SECRET`を設定する必要があります。また、ログイン時にリフレッシュトークンもセッションに保存するように、既存のログイン処理を修正する必要があります。

#### ステップ6：コントローラーを修正して、ジョブを投入する

最後に、`DLDWHDataObjectViewerController`を修正し、重い同期処理を直接実行する代わりに、作成したジョブをキューに投入するようにします。

**`DLDWHDataObjectViewerController.php` （`getModelData`のみ修正）**
```php
<?php

namespace App\HttpControllers;

use App\Models\DLDHWDataImportModel;
use App\Jobs\SyncBoxProject; // 作成したジョブをインポート
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class DLDWHDataObjectViewerController extends Controller
{
    public function getModelData(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        if (empty($projectFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }

        $dldwhModel = new DLDHWDataImportModel();
        
        // Boxにログインしているかチェック
        if (session()->has('access_token') && !empty(session('access_token'))) {
            // 【重要】同期ジョブをキューに投入する
            Log::info("Dispatching SyncBoxProject job for project: {$projectFolderId}");
            SyncBoxProject::dispatch(
                $projectFolderId,
                session('access_token'),
                session('refresh_token') // リフレッシュトークンも渡す
            );

            // ここでは同期の完了を待たずに、すぐにレスポンスを返すことも可能
            // 例えば、「同期を開始しました」というメッセージと共に、古いキャッシュを表示する
        }
        
        // 常にDBキャッシュからデータを取得して返す（ジョブはバックグラウンドで動く）
        $data = $dldwhModel->getCachedModelData($projectFolderId);

        if (isset($data['error'])) {
            return response()->json($data, 404);
        }
        return response()->json($data);
    }
    // ... その他の関数 ...
}
