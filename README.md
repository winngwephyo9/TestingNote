<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use GuzzleHttp\Client;
use GuzzleHttp\Pool; // <-- IMPORT THE POOL LIBRARY
use GuzzleHttp\Psr7\Request as GuzzleRequest; // <-- IMPORT TO AVOID NAMING CONFLICTS
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Exception;

class DLDWHDataObjectViewerController extends Controller
{
    public function objViewer()
    {
        return view('DLDWH.OBJViewer');
    }

    public function getCategoryNameByElementId(Request $request)
    {
        $WSCenID = $request->input('WSCenID');
        $elementIds = $request->input('ElementIds');
        $dldwhModle = new DLDHWDataImportModel();
        return $dldwhModle->getCategoryNameByElementId($WSCenID, $elementIds);
    }

    public function getProjectList(Request $request)
    {
        $mainFolderId = $request->input('folderId');
        if (!session()->has('access_token') || empty(session('access_token'))) {
            return response()->json(["no_token"]);
        }
        $access_token = session('access_token');

        if (empty($mainFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }

        try {
            $client = new Client(['verify' => false]);
            $requestURL = "https://api.box.com/2.0/folders/{$mainFolderId}/items?fields=id,name";
            $header = ["Authorization" => "Bearer " . $access_token];
            $response = $client->get($requestURL, ['headers' => $header]);
            $items = json_decode($response->getBody()->getContents())->entries;

            $projects = [];
            foreach ($items as $item) {
                if ($item->type == "folder") {
                    $projects[] = ['id' => $item->id, 'name' => $item->name];
                }
            }
            return response()->json($projects);
        } catch (Exception $e) {
            Log::error("Box API Error (getProjectList): " . $e->getMessage());
            return response()->json(['error' => "Failed to read project list from Box."], 500);
        }
    }

    public function getObjList(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        if (!session()->has('access_token') || empty(session('access_token'))) {
            return response()->json(['error' => 'Access token is missing.'], 401);
        }
        $accessToken = session('access_token');

        if (empty($projectFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }

        try {
            $client = new Client(['verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken];
            $allFolderItems = [];
            $offset = 0;
            $limit = 1000;

            do {
                $requestURL = "https://api.box.com/2.0/folders/{$projectFolderId}/items?fields=id,name&limit={$limit}&offset={$offset}";
                $response = $client->get($requestURL, ['headers' => $header]);
                $body = json_decode($response->getBody()->getContents());
                
                if (isset($body->entries)) {
                    $allFolderItems = array_merge($allFolderItems, $body->entries);
                    $offset += count($body->entries);
                } else {
                    break;
                }
            } while ($offset < $body->total_count);

            $objFiles = [];
            $mtlFile = null;

            foreach ($allFolderItems as $item) {
                if ($item->type == "file") {
                    $extension = strtolower(pathinfo($item->name, PATHINFO_EXTENSION));
                    if ($extension === 'mtl' && $mtlFile === null) {
                        $mtlFile = ['id' => $item->id, 'name' => $item->name];
                    } elseif ($extension === 'obj') {
                        $objFiles[] = ['id' => $item->id, 'name' => $item->name];
                    }
                }
            }

            if ($mtlFile === null) {
                return response()->json(['error' => 'No MTL file found in the folder.'], 404);
            }
            return response()->json(['mtl' => $mtlFile, 'objs' => $objFiles]);
        } catch (Exception $e) {
            Log::error("Box API Error (getObjList): " . $e->getMessage());
            return response()->json(['error' => "Failed to read file list from Box."], 500);
        }
    }

    /**
     * **FULLY SCALABLE AND ROBUST FUNCTION**
     * Gets download URLs for a large batch of files using a controlled concurrent pool.
     */
    public function getDownloadUrls(Request $request)
    {
        $fileIds = $request->input('fileIds');
        if (!session()->has('access_token') || empty(session('access_token'))) {
            return response()->json(['error' => 'Access token is missing.'], 401);
        }
        $accessToken = session('access_token');

        if (empty($fileIds) || !is_array($fileIds)) {
            return response()->json(['error' => 'File IDs array is required.'], 400);
        }

        try {
            $client = new Client(['verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken];
            $downloadUrls = [];

            // 1. Create a "generator" that will provide Guzzle with requests to run.
            $requests = function ($fileIds) use ($header) {
                foreach ($fileIds as $fileId) {
                    $url = "https://api.box.com/2.0/files/{$fileId}?fields=download_url";
                    // 'yield' is like 'return' but for generators. It provides one value at a time.
                    // We give each request an key ($fileId) so we know which one it was.
                    yield $fileId => new GuzzleRequest('GET', $url, $header);
                }
            };

            // 2. Create the Pool. This is the core of the solution.
            $pool = new Pool($client, $requests($fileIds), [
                // Set how many requests to run in parallel at any given time.
                'concurrency' => 25, // This is a safe and efficient number.
                
                // This function is called every time a request is successful.
                'fulfilled' => function ($response, $fileId) use (&$downloadUrls) {
                    // We use &$downloadUrls to modify the original array.
                    $fileInfo = json_decode($response->getBody()->getContents());
                    if (isset($fileInfo->download_url)) {
                        $downloadUrls[$fileId] = $fileInfo->download_url;
                    }
                },
                
                // This function is called every time a request fails.
                'rejected' => function ($reason, $fileId) {
                    Log::warning("Box API (Pool): Could not get download URL for file ID {$fileId}: " . $reason->getMessage());
                },
            ]);

            // 3. Initiate the transfers and wait for the pool to complete.
            $promise = $pool->promise();
            $promise->wait();

            return response()->json($downloadUrls);

        } catch (Exception $e) {
            Log::error("Box API Error (getDownloadUrls): " . $e->getMessage());
            return response()->json(['error' => "An error occurred while generating download URLs."], 500);
        }
    }
}
