<?php

namespace App\Models;

// ... use文 ...
use App\Models\ModelFileCache;

class DLDHWDataImportModel extends Model
{
    // ... 既存のすべてのメソッド ...

    /**
     * 【新規】DBキャッシュから、保存されているプロジェクトのリストを取得する
     * 
     * @return \Illuminate\Support\Collection
     */
    public function getCachedProjectList()
    {
        // model_file_cacheテーブルからproject_box_idとfile_nameを取得
        $projectsData = ModelFileCache::select('project_box_id', 'file_name')
            ->where('file_type', 'obj') // objファイルがあればプロジェクトとみなす
            ->get();

        // プロジェクトIDでグループ化し、最初のファイル名を取得してプロジェクト名とする
        // （より正確なプロジェクト名が必要な場合は、別途projectsテーブルが必要）
        $projects = $projectsData->groupBy('project_box_id')->map(function ($files, $projectId) {
            
            // プロジェクト名を取得するための仮のロジック
            // 本来はプロジェクト名もDBに保存するのが望ましい
            // ここでは、最初のOBJファイルのベース名からプロジェクト名を推測する
            $firstFileName = $files->first()->file_name;
            $projectName = pathinfo($firstFileName, PATHINFO_FILENAME); // 例: 240324_GF本社移転_...
            // よりクリーンな名前に加工することも可能
            // preg_match('/_([^_]+)_/', $projectName, $matches);
            // $projectName = $matches[1] ?? $projectName;


            return [
                'id' => $projectId,
                'name' => $projectName // 仮のプロジェクト名
            ];
        });

        return $projects->values(); // 連想配列を通常の配列に変換して返す
    }
}
```**注意:** このコードでは、プロジェクト名をファイル名から推測しています。より正確な運用のためには、`projects`という新しいテーブルを作り、`project_box_id`と`project_name`を保存するのが理想的です。

#### 2. `DLDWHDataObjectViewerController.php`を修正

次に、コントローラーの`getProjectList`メソッドを、新しく作ったモデルの機能を使うように修正します。

**`app/Http/Controllers/DLDWHDataObjectViewerController.php`**
```php
<?php

namespace App\Http\Controllers;

// ... use文 ...
use App\Models\DLDHWDataImportModel;

class DLDWHDataObjectViewerController extends Controller
{
    // ... objViewer, getModelData, getSyncStatus, getCategoryNameByElementId は変更なし ...
    
    /**
     * 【最終修正版】ログイン状態に応じて、BoxまたはDBキャッシュからプロジェクトリストを返す
     */
    public function getProjectList(Request $request)
    {
        $mainFolderId = $request->input('folderId');
        $dldwhModel = new DLDHWDataImportModel();

        // Boxにログインしているかチェック
        if (session()->has('access_token') && !empty(session('access_token'))) {
            // ==========================================================
            // ログイン時：Box APIから最新のプロジェクトリストを取得
            // ==========================================================
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
                
                return response()->json([
                    'login_status' => 'logged_in',
                    'projects' => $projects
                ]);

            } catch (Exception $e) {
                Log::error("Failed to read project list from Box: " . $e->getMessage());
                // Boxとの通信に失敗した場合でも、フォールバックとしてDBキャッシュを試みる
                $cachedProjects = $dldwhModel->getCachedProjectList();
                return response()->json([
                    'login_status' => 'error',
                    'projects' => $cachedProjects,
                    'message' => "Failed to connect to Box. Displaying cached projects."
                ]);
            }

        } else {
            // ==========================================================
            // 未ログイン時：DBキャッシュからプロジェクトリストを取得
            // ==========================================================
            $cachedProjects = $dldwhModel->getCachedProjectList();
            
            return response()->json([
                'login_status' => 'logged_out',
                'projects' => $cachedProjects
            ]);
        }
    }
}




 public function getProjectList(Request $request)
    {
        $mainFolderId = $request->input('folderId');
        $dldwhModel = new DLDHWDataImportModel();

        // Boxにログインしているかチェック
        if (session()->has('access_token') && !empty(session('access_token'))) {
            // ==========================================================
            // ログイン時：Box APIから最新のプロジェクトリストを取得
            // ==========================================================
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
                
                return response()->json([
                    'login_status' => 'logged_in',
                    'projects' => $projects
                ]);

            } catch (Exception $e) {
                Log::error("Failed to read project list from Box: " . $e->getMessage());
                // Boxとの通信に失敗した場合でも、フォールバックとしてDBキャッシュを試みる
                $cachedProjects = $dldwhModel->getCachedProjectList();
                return response()->json([
                    'login_status' => 'error',
                    'projects' => $cachedProjects,
                    'message' => "Failed to connect to Box. Displaying cached projects."
                ]);
            }

        } else {
            // ==========================================================
            // 未ログイン時：DBキャッシュからプロジェクトリストを取得
            // ==========================================================
            $cachedProjects = $dldwhModel->getCachedProjectList();
            
            return response()->json([
                'login_status' => 'logged_out',
                'projects' => $cachedProjects
            ]);
        }
