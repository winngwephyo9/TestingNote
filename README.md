<?php

namespace App\Models;

use App\Models\ModelFileCache;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Http;
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Exception\ClientException;
use Illuminate\Support\Carbon;
use Exception;
use Throwable;

class DLDHWDataImportModel extends Model
{
    public function getCategoryNameByElementId($WSCenID, $elementIds)
    {
        // ... (このメソッドは変更なし) ...
    }
    
    public function getCachedProjectList()
    {
        $projects = ModelFileCache::select('project_box_id', 'project_name')
            ->whereNotNull('project_name')->distinct()->get()
            ->map(function ($file) {
                return ['id' => $file->project_box_id, 'name' => $file->project_name];
            });
        return $projects->sortBy('name')->values();
    }

    public function getCachedModelData($projectFolderId)
    {
        $cachedFiles = ModelFileCache::where('project_box_id', $projectFolderId)->get()->groupBy('base_name');
        if ($cachedFiles->isEmpty()) {
            return ['error' => 'No cached model found. Please log in to Box to sync data.'];
        }
        return $this->formatDataForFrontend($cachedFiles);
    }
    
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
                $updateChunks = array_chunk($filesToUpdate, 200);
                foreach ($updateChunks as $chunkOfFiles) {
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
                                'created_at' => now(), 'updated_at' => now(),
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

    private function fetchFullBoxFileList($projectFolderId, $accessToken, $client)
    {
        $header = ["Authorization" => "Bearer " . $accessToken];
        $allFolderItems = []; $offset = 0; $limit = 1000;
        do {
            $requestURL = "https://api.box.com/2.0/folders/{$projectFolderId}/items?fields=id,name,modified_at&limit={$limit}&offset={$offset}";
            $response = $client->get($requestURL, ['headers' => $header]);
            $body = json_decode($response->getBody()->getContents(), true);
            if (isset($body['entries'])) {
                $allFolderItems = array_merge($allFolderItems, $body['entries']);
                $offset += count($body['entries']);
            } else { break; }
        } while (isset($body['total_count']) && $offset < $body['total_count']);
        return $allFolderItems;
    }
    
    private function fetchMultipleBoxFileContentsConcurrently($files, $client)
    {
        $downloadedContents = []; $tokenExpired = false;
        $header = ["Authorization" => "Bearer " . session('access_token')];
        $requests = function ($files) use ($header) {
            foreach ($files as $file) {
                $url = "https://api.box.com/2.0/files/{$file['id']}/content";
                yield $file['id'] => new \GuzzleHttp\Psr7\Request('GET', $url, $header);
            }
        };
        $pool = new Pool($client, $requests($files), [
            'concurrency' => 10,
            'fulfilled' => function ($response, $fileId) use (&$downloadedContents) { $downloadedContents[$fileId] = $response->getBody()->getContents(); },
            'rejected' => function ($reason, $fileId) { Log::error("Failed to download file ID {$fileId}: " . $reason->getMessage()); }
        ]);
        $promise = $pool->promise(); $promise->wait();
        return ['status' => 'success', 'contents' => $downloadedContents];
    }
    
    public function refreshBoxToken($refreshToken, $urlPath)
    {
        if (empty($refreshToken)) { Log::error("Refresh token is empty."); return null; }
        try {
            $clientId = env("BOX_CLIENT_ID"); $clientSecret = env("BOX_CLIENT_SECRET");
            if (strpos($urlPath, 'deployment') !== false) {
                $clientId = env("BOX_CLIENT_ID_FOR_DEV_SLOT"); $clientSecret = env("BOX_CLIENT_SECRET_FOR_DEV_SLOT");
            }
            if (!$clientId || !$clientSecret) { Log::error("Box client credentials are not set."); return null; }
            $response = Http::asForm()->post('https://api.box.com/oauth2/token', [
                'grant_type' => 'refresh_token', 'refresh_token' => $refreshToken,
                'client_id' => $clientId, 'client_secret' => $clientSecret,
            ]);
            if ($response->successful()) {
                $data = $response->json();
                Log::info("Successfully refreshed Box tokens."); $this->saveBoxExpiryStatus();
                return ['access_token' => $data['access_token'], 'refresh_token' => $data['refresh_token']];
            } else {
                Log::error("Failed to refresh Box token.", ['status' => $response->status(), 'body' => $response->body()]);
                return null;
            }
        } catch (Exception $e) { Log::error("Exception on refresh token: " . $e->getMessage()); return null; }
    }
    
    public function saveBoxExpiryStatus()
    {
        try { DB::connection('dldwh')->table('box_expiry_status')->insert(['status' => 'true', 'created_at' => now()]); }
        catch (Exception $e) { Log::error("Failed to save box expiry status: " . $e->getMessage()); }
    }
}
