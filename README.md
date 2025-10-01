<?php

namespace App\Models;

use App\Models\ModelFileCache;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
// ... 他のuse文 ...
use Throwable;

class DLDHWDataImportModel extends Model
{
    // ... クラスプロパティの宣言 ...
    protected $accessToken;
    protected $refreshToken;
    // ...

    // ... コンストラクタ ...

    /**
     * 【最終修正版】メモリ効率とタイムアウトを解決する同期ロジック
     */
    public function syncAndGetModelData($projectFolderId, $accessToken, $refreshToken, $urlPath, $boxLoginTime)
    {
        mb_internal_encoding('UTF-8');
        set_time_limit(0);
        
        // ジョブから渡された情報をクラスのプロパティに保存
        $this->accessToken = $accessToken;
        $this->refreshToken = $refreshToken;
        $this->urlPath = $urlPath;
        $this->boxLoginTime = $boxLoginTime;
        
        try {
            $client = new Client(['verify' => false]);
            $projectName = $this->fetchProjectName($projectFolderId, $client);

            // DBに保存されている全ファイルIDを一度だけ取得（差分チェックのため）
            $dbFiles = ModelFileFileCache::where('project_box_id', $projectFolderId)->get()->keyBy('box_file_id');
            
            // =================================================================
            //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
            //
            //  **【重要】ファイルリストの取得と処理をページ単位でループ実行する**
            //
            $offset = 0;
            $limit = 1000;
            $totalFilesProcessed = 0;
            $totalFilesUpdated = 0;

            do {
                // 1. Boxからファイルリストを1ページ(1000件)だけ取得
                $pageOfFilesResponse = $this->fetchBoxFileListByPage($projectFolderId, $client, $limit, $offset);
                $pageOfFiles = $pageOfFilesResponse['entries'];

                if (empty($pageOfFiles)) break; // 取得するファイルがなくなったらループを抜ける

                Log::info("Processing file list page from offset {$offset}. Files in page: " . count($pageOfFiles));
                
                // 2. この1000件の中から、更新が必要なファイルだけを抽出
                $filesToUpdateInPage = [];
                foreach ($pageOfFiles as $boxFile) {
                    $dbFile = $dbFiles->get($boxFile['id']);
                    $boxModifiedAtJst = Carbon::parse($boxFile['modified_at'])->setTimezone('Asia/Tokyo')->startOfSecond();
                    $dbModifiedAtJst = $dbFile ? $dbFile->box_modified_at->startOfSecond() : null;
                    if (!$dbFile || $dbModifiedAtJst->lt($boxModifiedAtJst)) {
                        $filesToUpdateInPage[] = $boxFile;
                    }
                }
                
                // 3. 更新が必要なファイルがあれば、ダウンロードとDB保存を実行
                if (!empty($filesToUpdateInPage)) {
                    Log::info("Found " . count($filesToUpdateInPage) . " files to update in this page.");
                    
                    // fetchContentsWithRefreshは内部でリフレッシュを処理し、最新のトークンを返す
                    $downloadedContents = $this->fetchContentsWithRefresh($filesToUpdateInPage);
                    
                    if (!empty($downloadedContents)) {
                        $boxFilesByIdInPage = collect($filesToUpdateInPage)->keyBy('id');
                        $dataToUpsert = [];
                        foreach ($downloadedContents as $fileId => $content) {
                            $boxFile = $boxFilesByIdInPage->get($fileId);
                            if ($boxFile) {
                                $dataToUpsert[] = [ /* ... contentを含むデータ配列 ... */ ];
                            }
                        }
                        if (!empty($dataToUpsert)) {
                            ModelFileCache::upsert($dataToUpsert, ['box_file_id'], [/* ... */]);
                        }
                    }
                    $totalFilesUpdated += count($filesToUpdateInPage);
                }
                
                $offset += count($pageOfFiles);
                $totalFilesProcessed += count($pageOfFiles);
                
            } while ($totalFilesProcessed < $pageOfFilesResponse['total_count']);
            //
            //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
            // =================================================================

            // Boxに存在しなくなったファイルをDBから削除する処理は、最後に一度だけ行う
            $allBoxFileIds = $this->fetchAllBoxFileIds($projectFolderId, $client);
            $this->cleanupDeletedFiles($projectFolderId, $allBoxFileIds);

            return $totalFilesUpdated;

        } catch (Throwable $e) {
            Log::error("Box Sync Failed: " . $e->getMessage(), ['exception' => $e]);
            throw $e;
        }
    }
    
    /**
     * 【新規・修正】Box APIからファイルリストをページ単位で取得する
     */
    private function fetchBoxFileListByPage($folderId, $client, $limit, $offset)
    {
        $this->checkBoxExpiryStatus(); // APIを叩く前にトークンをチェック
        $header = ["Authorization" => "Bearer " . $this->accessToken];
        $requestURL = "https://api.box.com/2.0/folders/{$folderId}/items?fields=id,name,modified_at&limit={$limit}&offset={$offset}";
        $response = $client->get($requestURL, ['headers' => $header]);
        return json_decode($response->getBody()->getContents(), true);
    }
    
    /**
     * 【新規】Boxから全ファイルIDだけを効率的に取得する（削除チェック用）
     */
    private function fetchAllBoxFileIds($folderId, $client)
    {
        $allFileIds = [];
        // ... fields=id だけを指定して、ページネーションループで全IDを取得するロジック ...
        return $allFileIds;
    }
    
    /**
     * 【修正版】ダウンロードとリフレッシュのラッパー
     */
    private function fetchContentsWithRefresh(array $filesToDownload)
    {
        $maxRetries = 3;
        for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
            $result = $this->fetchMultipleBoxFileContentsConcurrently($filesToDownload);
            if ($result['status'] === 'success') {
                return $result['contents'];
            }
            if ($result['status'] === 'token_expired') {
                $this->checkBoxExpiryStatus(); // リフレッシュを実行
                continue;
            }
            throw new Exception($result['message'] ?? 'Unknown download error.');
        }
        throw new Exception("Failed to download files after {$maxRetries} refresh attempts.");
    }

    /**
     * 【修正版】並列ダウンロード関数は$this->accessTokenを参照
     */
    private function fetchMultipleBoxFileContentsConcurrently(array $files)
    {
        $header = ["Authorization" => "Bearer " . $this->accessToken];
        // ... (以前の回答と同じ、Poolのロジック) ...
    }

    // ... wnpcheckBoxExpiryStatus, wnpsaveAccessTokenなどのメソッドは、
    // このクラスの標準メソッドとしてリファクタリング・統一する ...
}
