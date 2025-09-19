CREATE TABLE `box_tokens` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `access_token` text COLLATE utf8mb4_unicode_ci NOT NULL,
  `refresh_token` text COLLATE utf8mb4_unicode_ci NOT NULL,
  `login_time` int(11) NOT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;


// DLDHWDataImportModel.php

class DLDHWDataImportModel extends Model
{
    // ... 既存のメソッド ...

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
    
    /**
     * 【修正版】リフレッシュ後にDBとセッションの両方を更新する
     */
    public function refreshBoxToken($refreshToken, $urlPath)
    {
        // ... (APIリクエストのロジックは同じ) ...
        
        if ($response->successful()) {
            $data = $response->json();
            $newAccessToken = $data['access_token'];
            $newRefreshToken = $data['refresh_token'];

            // 【重要】DBとセッションの両方を更新
            $this->saveBoxTokens($newAccessToken, $newRefreshToken);
            session(['access_token' => $newAccessToken, 'refresh_token' => $newRefreshToken, 'box_login_time' => time()]);

            Log::info("Successfully refreshed and saved Box tokens.");
            return ['access_token' => $newAccessToken, 'refresh_token' => $newRefreshToken];
        }
        
        // ... (失敗時のロジック) ...
    }
}


// DLDWHDataObjectViewerController.php

class DLDWHDataObjectViewerController extends Controller
{
    // ... 他のメソッド ...

    /**
     * 【修正版】Box認証後にDBとセッションにトークンを保存する
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
}```

#### ステップ4：ジョブの修正 (`app/Jobs/SyncBoxProject.php`)

ジョブの開始時に、コンストラクタで渡された古い情報ではなく、**必ずDBから最新のトークンを読み込む**ようにします。

```php
// app/Jobs/SyncBoxProject.php

class SyncBoxProject implements ShouldQueue
{
    protected $projectFolderId;
    // 【重要】セッションデータはもうコンストラクタで受け取らない

    public function __construct($projectFolderId) // コンストラクタをシンプルにする
    {
        $this->projectFolderId = $projectFolderId;
    }

    public function handle()
    {
        try {
            // 【重要】ジョブ開始時にDBから最新のトークン情報を取得
            $dldwhModel = new DLDHWDataImportModel();
            $latestTokens = $dldwhModel->getLatestBoxTokens();
            if (!$latestTokens) {
                throw new Exception("No valid tokens found in the database.");
            }
            
            // 取得した最新情報をセッションにセットして、後続の処理で使えるようにする
            session([
                'access_token' => $latestTokens->access_token,
                'refresh_token' => $latestTokens->refresh_token,
                'box_login_time' => $latestTokens->login_time,
                'url_path' => url('/') // url_pathはここで設定
            ]);
            
            $this->checkBoxExpiryStatus(); // トークンチェックを実行
            
            // ... (syncAndGetModelDataの呼び出しと完了/失敗の記録) ...

        } catch (Throwable $e) { /* ... */ }
    }
    
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

    // ... failed()メソッド ...
}


class SyncBoxProject implements ShouldQueue
{
    protected $projectFolderId;
    // 【重要】セッションデータはもうコンストラクタで受け取らない

    public function __construct($projectFolderId) // コンストラクタをシンプルにする
    {
        $this->projectFolderId = $projectFolderId;
    }

    public function handle()
    {
        try {
            // 【重要】ジョブ開始時にDBから最新のトークン情報を取得
            $dldwhModel = new DLDHWDataImportModel();
            $latestTokens = $dldwhModel->getLatestBoxTokens();
            if (!$latestTokens) {
                throw new Exception("No valid tokens found in the database.");
            }
            
            // 取得した最新情報をセッションにセットして、後続の処理で使えるようにする
            session([
                'access_token' => $latestTokens->access_token,
                'refresh_token' => $latestTokens->refresh_token,
                'box_login_time' => $latestTokens->login_time,
                'url_path' => url('/') // url_pathはここで設定
            ]);
            
            $this->checkBoxExpiryStatus(); // トークンチェックを実行
            
            // ... (syncAndGetModelDataの呼び出しと完了/失敗の記録) ...

        } catch (Throwable $e) { /* ... */ }
    }
    
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

    // ... failed()メソッド ...
}



#### ステップ5：JavaScriptの修正 (`objViewerStandard.js`)

エラー時にローディングが消えないように、`catch`ブロックを修正します。

```javascript
// objViewerStandard.js の checkSyncStatus 関数

function checkSyncStatus(projectFolderId, projectName) {
    setTimeout(async () => {
        // ...
        try {
            // ... (ajaxリクエスト) ...

            if (response.status === 'completed' || response.status === 'no_files_to_update') {
                // ...
            } else if (response.status === 'failed') {
                // ...
            } else { // processing
                // ...
            }
        } catch (error) {
            // 【重要】エラー時にローディングを消さず、メッセージを表示してポーリングを続ける
            console.error("Failed to check sync status:", error);
            if (loaderTextElement) loaderTextElement.textContent = `状態の確認に失敗しました。数秒後に再試行します...`;
            // ポーリングを継続
            checkSyncStatus(projectFolderId, projectName); 
        }
    }, 5000);
}





<img width="1240" height="873" alt="image" src="https://github.com/user-attachments/assets/b77c4c25-1cd8-46a2-a319-4236586773a9" />

