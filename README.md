<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use GuzzleHttp\Client;
use GuzzleHttp\HandlerStack; // <-- IMPORT HandlerStack
use GuzzleHttp\Middleware; // <-- IMPORT Middleware
use GuzzleHttp\Pool;
use GuzzleHttp\Psr7\Request as GuzzleRequest;
use GuzzleHttp\Psr7\Response; // <-- IMPORT Response
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
     * **PRODUCTION-READY, RATE-LIMIT-PROOF FUNCTION**
     * Gets download URLs using a controlled pool WITH automatic retries on failure.
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
            // 1. Create a HandlerStack
            $stack = HandlerStack::create();

            // 2. Create the retry middleware
            $stack->push(Middleware::retry(
                // The first function decides IF a retry should happen
                function ($retries, GuzzleRequest $request, Response $response = null, $exception = null) {
                    // Limit to 5 retries
                    if ($retries >= 5) {
                        return false;
                    }
                    // Retry on server errors (5xx) or rate limit errors (429)
                    if ($response && in_array($response->getStatusCode(), [429, 500, 502, 503, 504])) {
                         Log::warning("Retrying request. Attempt: {$retries}, Status: {$response->getStatusCode()}");
                        return true;
                    }
                    return false;
                },
                // The second function decides HOW LONG to wait before retrying
                function ($retries, Response $response) {
                    // If the Box API tells us how long to wait, respect it.
                    if ($response->hasHeader('Retry-After')) {
                        return (int)$response->getHeader('Retry-After')[0];
                    }
                    // Otherwise, use exponential backoff (1s, 2s, 4s, 8s) with a little randomness
                    return (int)pow(2, $retries) * 1000 + rand(0, 100);
                }
            ));

            // 3. Create a client with our new retry-enabled handler
            $client = new Client(['handler' => $stack, 'verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken];
            $downloadUrls = [];

            // 4. Create the request generator
            $requests = function ($fileIds) use ($header) {
                foreach ($fileIds as $fileId) {
                    $url = "https://api.box.com/2.0/files/{$fileId}?fields=download_url";
                    yield $fileId => new GuzzleRequest('GET', $url, $header);
                }
            };

            // 5. Create the Pool with a more conservative concurrency
            $pool = new Pool($client, $requests($fileIds), [
                'concurrency' => 10, // Reduced from 25 to be less aggressive
                'fulfilled' => function ($response, $fileId) use (&$downloadUrls) {
                    $fileInfo = json_decode($response->getBody()->getContents());
                    if (isset($fileInfo->download_url)) {
                        $downloadUrls[$fileId] = $fileInfo->download_url;
                    }
                },
                'rejected' => function ($reason, $fileId) {
                    Log::error("Box API (Pool): Request failed definitively for file ID {$fileId}: " . $reason->getMessage());
                },
            ]);

            // 6. Run the pool
            $promise = $pool->promise();
            $promise->wait();

            return response()->json($downloadUrls);

        } catch (Exception $e) {
            Log::error("Critical Error in getDownloadUrls: " . $e->getMessage());
            return response()->json(['error' => "A critical error occurred while generating download URLs."], 500);
        }
    }
}
