<?php

namespace App\Models;

// ... 他に必要なuse文 ...
use GuzzleHttp\Client;
use GuzzleHttp\Exception\ClientException;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Middleware;
use GuzzleHttp\Promise\EachPromise;
use GuzzleHttp\Psr7\Request as GuzzleRequest;
use GuzzleHttp\Psr7\Response;
use Illuminate-Support-Facades-Log;
use Exception;

class DLDHWDataImportModel extends Model
{
    // ... getCachedModelData, syncAndGetModelData, formatDataForFrontend, fetchFullBoxFileList は変更ありません ...
    
    /**
     * 【修正版】トークン切れに対応し、自動リフレッシュとリトライを行う堅牢な並列ダウンロード関数
     */
    private function fetchMultipleBoxFileContentsConcurrently($files)
    {
        $downloadedContents = [];
        $tokenExpiredAndRefreshed = false; // トークンを一度リフレッシュしたかどうかのフラグ

        // 無限ループを防ぐための試行回数カウンター
        $maxRetries = 2; 
        $attempt = 1;

        while ($attempt <= $maxRetries) {
            
            // 常にセッションから最新のアクセストークンを取得
            $accessToken = session('access_token');
            if (empty($accessToken)) {
                Log::error("No access token available for download.");
                return ['status' => 'error', 'message' => 'No access token.', 'contents' => []];
            }

            $tokenExpiredInThisAttempt = false; // この試行でトークンが切れたかどうかのフラグ
            
            $client = new Client(['verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken];

            // ダウンロードが成功していないファイルのリストを毎回作成
            $remainingFiles = array_filter($files, function ($file) use ($downloadedContents) {
                return !isset($downloadedContents[$file->id]);
            });

            if (empty($remainingFiles)) {
                break; // ダウンロードが全て完了したらループを抜ける
            }

            Log::info("Download attempt #{$attempt}. Remaining files: " . count($remainingFiles));

            $requests = function ($files) use ($header) {
                foreach ($files as $file) {
                    $url = "https://api.box.com/2.0/files/{$file->id}/content";
                    yield $file->id => new GuzzleRequest('GET', $url, $header);
                }
            };
            
            $promise = (new EachPromise($requests($remainingFiles), [
                'concurrency' => 10,
                'fulfilled' => function ($response, $fileId) use (&$downloadedContents) {
                    $downloadedContents[$fileId] = $response->getBody()->getContents();
                },
                'rejected' => function ($reason, $fileId) use (&$tokenExpiredInThisAttempt, &$promise) {
                    if ($reason instanceof ClientException && $reason->getResponse()->getStatusCode() == 401) {
                        Log::warning("Box token expired (401) during download of file ID {$fileId}.");
                        $tokenExpiredInThisAttempt = true; // この試行でトークンが切れたことを記録
                        if (method_exists($promise, 'cancel')) {
                            $promise->cancel(); // 残りのダウンロードをキャンセルして、速やかにループの次のステップに進む
                        }
                    } else {
                        Log::error("Failed to download content for file ID {$fileId}: " . $reason->getMessage());
                    }
                }
            ]))->promise();

            try {
                $promise->wait();
            } catch(Exception $e) {
                 Log::error("An unexpected error occurred during Guzzle execution: " . $e->getMessage());
                 return ['status' => 'error', 'message' => $e->getMessage(), 'contents' => []];
            }
            
            // --- ループの最後にトークン切れをチェック ---
            if ($tokenExpiredInThisAttempt) {
                Log::info("Attempting to refresh Box token...");
                $newAccessToken = $this->refreshBoxToken();

                if (!$newAccessToken) {
                    Log::error("Failed to refresh Box token. Aborting sync process.");
                    return ['status' => 'error', 'message' => 'Failed to refresh token.', 'contents' => $downloadedContents];
                }
                
                Log::info("Token refreshed successfully. Continuing download process.");
                $attempt++; // 次の試行へ
                continue; // ループの最初に戻って、残りのファイルのダウンロードを再開
            }
            
            // トークン切れが発生しなかった場合、正常に終了
            break;
        }

        return ['status' => 'success', 'contents' => $downloadedContents];
    }
    
    /**
     * 【新規】Boxのアクセストークンをリフレッシュするヘルパーメソッド
     */
    private function refreshBoxToken()
    {
        try {
            // **重要**: ジョブ内でセッションを扱う場合、ヘルパー関数を使うのが安全
            $refreshToken = session('refresh_token');
            
            if (!$refreshToken) {
                Log::error("No refresh token found in session.");
                return null;
            }

            // .envファイルからクライアントIDとシークレットを取得
            $clientId = env('BOX_CLIENT_ID');
            $clientSecret = env('BOX_CLIENT_SECRET');

            if (!$clientId || !$clientSecret) {
                Log::error("BOX_CLIENT_ID or BOX_CLIENT_SECRET is not set in .env file.");
                return null;
            }

            $client = new Client(['verify' => false]);
            $response = $client->post('https://api.box.com/oauth2/token', [
                'form_params' => [
                    'grant_type' => 'refresh_token',
                    'refresh_token' => $refreshToken,
                    'client_id' => $clientId,
                    'client_secret' => $clientSecret,
                ]
            ]);

            $data = json_decode($response->getBody()->getContents());
            
            if (isset($data->access_token) && isset($data->refresh_token)) {
                // 新しいトークンをセッションに保存
                session(['access_token' => $data->access_token]);
                session(['refresh_token' => $data->refresh_token]);
                
                Log::info("Successfully refreshed Box tokens.");
                return $data->access_token;
            } else {
                Log::error("Refresh token response did not contain new tokens.");
                return null;
            }

        } catch (Exception $e) {
            Log::error("Exception occurred while refreshing Box token: " . $e->getMessage());
            return null;
        }
    }
}
