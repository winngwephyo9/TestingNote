<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Psr7\Request as GuzzleRequest;
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
        if (session()->has('access_token')) {
            $access_token = session('access_token');
            if ($access_token == "") {
                return "no_token";
            }

            if (empty($mainFolderId) || empty($access_token)) {
                return response()->json(['error' => 'Folder ID and Access Token are required.'], 400);
            }

            try {
                $client = new Client(['verify' => false]);
                $requestURL = "https://api.box.com/2.0/folders/" . $mainFolderId . "/items?fields=id,name";
                $header = [
                    "Authorization" => "Bearer " . $access_token,
                    "Accept" => "application/json"
                ];

                $response = $client->request('GET', $requestURL, ['headers' => $header]);
                $items = json_decode($response->getBody()->getContents())->entries;

                $projects = [];
                foreach ($items as $item) {
                    if ($item->type == "folder") {
                        $projects[] = ['id' => $item->id, 'name' => $item->name];
                    }
                }
                return response()->json($projects);
            } catch (Exception $e) {
                return response()->json(['error' => "Failed to read project list from Box: " . $e->getMessage()], 500);
            }
        } else {
            return "no_token";
        }
    }

    public function getObjList(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        $accessToken = session('access_token');

        if (empty($projectFolderId) || empty($accessToken)) {
            return response()->json(['error' => 'Folder ID and Access Token are required.'], 400);
        }

        try {
            $client = new Client(['verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken, "Accept" => "application/json"];

            $limit = 1000;
            $offset = 0;
            $allFolderItems = [];

            do {
                $requestURL = "https://api.box.com/2.0/folders/" . $projectFolderId . "/items?fields=id,name&limit=" . $limit . "&offset=" . $offset;
                $response = $client->request('GET', $requestURL, ['headers' => $header]);
                $body = json_decode($response->getBody()->getContents());

                $allFolderItems = array_merge($allFolderItems, $body->entries);
                $offset += $limit;
            } while (count($allFolderItems) < $body->total_count);

            $objFiles = [];
            $mtlFile = null;

            foreach ($allFolderItems as $item) {
                if ($item->type == "file" && strtolower(pathinfo($item->name, PATHINFO_EXTENSION)) === 'mtl') {
                    $mtlFile = ['id' => $item->id, 'name' => $item->name];
                    break;
                }
            }

            if ($mtlFile === null) {
                return response()->json(['error' => 'No MTL file found in the specified folder.'], 404);
            }

            foreach ($allFolderItems as $item) {
                if ($item->type == "file" && strtolower(pathinfo($item->name, PATHINFO_EXTENSION)) === 'obj') {
                    $objFiles[] = ['id' => $item->id, 'name' => $item->name];
                }
            }

            return response()->json(['mtl' => $mtlFile, 'objs' => $objFiles]);
        } catch (Exception $e) {
            return response()->json(['error' => "Failed to read file list from Box: " . $e->getMessage()], 500);
        }
    }

    /**
     * MODIFIED FUNCTION: Gets temporary, pre-authenticated download URLs for multiple files from Box concurrently.
     */
    public function getDownloadUrls(Request $request)
    {
        $fileIds = $request->input('fileIds');
        $accessToken = session('access_token');

        if (empty($fileIds) || !is_array($fileIds) || empty($accessToken)) {
            return response()->json(['error' => 'File IDs array and Access Token are required.'], 400);
        }

        $client = new Client(['verify' => false]);
        $downloadUrls = [];

        // Define a generator that creates all the requests we need to send
        $requests = function ($fileIds, $accessToken) {
            $header = [
                "Authorization" => "Bearer " . $accessToken,
                "Accept" => "application/json"
            ];
            foreach ($fileIds as $fileId) {
                $url = "https://api.box.com/2.0/files/" . $fileId . "?fields=download_url";
                // Yield the fileId along with the request to identify it later
                yield $fileId => new GuzzleRequest('GET', $url, $header);
            }
        };

        $pool = new Pool($client, $requests($fileIds, $accessToken), [
            'concurrency' => 20, // Number of parallel requests. Adjust as needed.
            'fulfilled' => function ($response, $fileId) use (&$downloadUrls) {
                // This is called when a request is successful
                $fileInfo = json_decode($response->getBody()->getContents());
                if (isset($fileInfo->download_url)) {
                    $downloadUrls[$fileId] = $fileInfo->download_url;
                }
            },
            'rejected' => function ($reason, $fileId) {
                // This is called when a request fails
                Log::error("Failed to get download URL for file ID {$fileId}: " . $reason->getMessage());
            },
        ]);

        // Initiate the transfers and wait for them to complete
        $promise = $pool->promise();
        $promise->wait();

        return response()->json($downloadUrls);
    }
}
