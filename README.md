<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use GuzzleHttp\Client;
use GuzzleHttp\Cookie\CookieJar;
use GuzzleHttp\Exception\GuzzleException;
use Illuminate\Http\Request; // For API request
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\File;
use App\Models\LoginModel;
use Exception;
use GuzzleHttp\Exception\ClientException;

class DLDWHDataObjectViewerController extends Controller
{
    public function objViewer()
    {
        return view('DLDWH.OBJViewer');
    }

    /**
     * get category name by elementId
     * @param $request
     * @return string
     */
    public function getCategoryNameByElementId(Request $request)
    {
        $WSCenID = $request->input('WSCenID');
        $elementIds = $request->input('ElementIds');
        $dldwhModle = new DLDHWDataImportModel();
        return $dldwhModle->getCategoryNameByElementId($WSCenID, $elementIds);
    }

    /**
     * NEW FUNCTION: Lists the sub-folders (projects) within a main Box folder.
     * This will populate the dropdown menu.
     */
    public function getProjectList(Request $request)
    {
        $mainFolderId = $request->input('folderId');
        if (session()->has('access_token')) {
            $access_token = session('access_token');
            if ($access_token == "") {
                return "no_token";
            }
            if (session()->has('authority_id')) {
                $login = new LoginModel();
                $result = $login->GetBoxAuthority(session('authority_id'));
                // $mainFolderId = "332324771912"; // folderId
                // $this->getProjectListInfo($mainFolderId, $access_token);
                // return "success";
                log::info("FolderId : " . $mainFolderId);
                log::info("accessToken : " . $access_token);

                if (empty($mainFolderId) || empty($access_token)) {
                    return response()->json(['error' => 'Folder ID and Access Token are required.'], 400);
                }

                try {
                    $client = new Client(['verify' => false]); // Consider setting 'verify' to true in production with proper SSL certs
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
                            $projects[] = [
                                'id' => $item->id,       // The Folder ID (e.g., "12345" for "GF")
                                'name' => $item->name,   // The Folder Name (e.g., "GF")
                            ];
                        }
                    }
                    return response()->json($projects);
                } catch (Exception $e) {
                    return response()->json(['error' => "Failed to read project list from Box: " . $e->getMessage()], 500);
                }
            }
        } else {
            return "no_token";
        }
    }

    /**
     * MODIFIED FUNCTION: Lists ALL OBJ/MTL files within a specific project folder on Box,
     * handling pagination automatically.
     */
    public function getObjList(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        if (session()->has('access_token')) {
            $accessToken = session('access_token');
        }
        if (empty($projectFolderId) || empty($accessToken)) {
            return response()->json(['error' => 'Folder ID and Access Token are required.'], 400);
        }

        try {
            $client = new Client(['verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken, "Accept" => "application/json"];

            $limit = 1000;
            $offset = 0;
            $totalCount = 0;
            $allFolderItems = [];

            do {
                $requestURL = "https://api.box.com/2.0/folders/" . $projectFolderId . "/items?fields=id,name&limit=" . $limit . "&offset=" . $offset;
                $response = $client->request('GET', $requestURL, ['headers' => $header]);
                $body = json_decode($response->getBody()->getContents());

                if ($offset === 0) $totalCount = $body->total_count;
                $allFolderItems = array_merge($allFolderItems, $body->entries);
                $offset += $limit;
            } while (count($allFolderItems) < $totalCount);

            $objFiles = [];
            $mtlFile = null;

            // First, find the single MTL file in the folder
            foreach ($allFolderItems as $item) {
                if ($item->type == "file" && strtolower(pathinfo($item->name, PATHINFO_EXTENSION)) === 'mtl') {
                    $mtlFile = ['id' => $item->id, 'name' => $item->name];
                    break; // Assume there's only one
                }
            }

            if ($mtlFile === null) {
                return response()->json(['error' => 'No MTL file found in the specified folder.'], 404);
            }

            // Now, collect all OBJ files
            foreach ($allFolderItems as $item) {
                if ($item->type == "file" && strtolower(pathinfo($item->name, PATHINFO_EXTENSION)) === 'obj') {
                    $objFiles[] = ['id' => $item->id, 'name' => $item->name];
                }
            }

            return response()->json([
                'mtl' => $mtlFile,
                'objs' => $objFiles
            ]);
        } catch (Exception $e) {
            return response()->json(['error' => "Failed to read file list from Box: " . $e->getMessage()], 500);
        }
    }


    public function getFileContents(Request $request)
    {
        $fileId = $request->input('fileId');
        $access_token = session('access_token');

        if (empty($fileId) || empty($access_token)) {
            return response()->json(['error' => 'File ID or Access Token is missing.'], 400);
        }

        try {
            $client = new Client(['verify' => false]);
            $requestURL = "https://api.box.com/2.0/files/" . $fileId . "/content";
            $header = ["Authorization" => "Bearer " . $access_token];

            $response = $client->request('GET', $requestURL, ['headers' => $header]);

            // Return the raw file content directly
            return $response->getBody()->getContents();
            // $fileContent = $response->getBody()->getContents();
            // return response($fileContent, 200)->header('Content-Type', 'text/plain');
        } catch (ClientException $e) {
            // --- THIS IS THE KEY IMPROVEMENT ---
            // This specifically catches HTTP errors from the API call (like 401 Unauthorized)
            $statusCode = $e->getResponse()->getStatusCode();
            $boxErrorBody = $e->getResponse()->getBody()->getContents();
            log::error("Box API ClientException for file ID {$fileId}: " . $boxErrorBody);

            // Return a specific error message to the frontend
            return response()->json([
                'error' => 'Box API Error',
                'status' => $statusCode,
                'message' => 'Failed to fetch file from Box. The access token may have expired or permissions are insufficient.',
                'box_response' => json_decode($boxErrorBody) // Send the actual Box error back
            ], $statusCode); // Use the status code from Box

        } catch (Exception $e) {
            log::error("Generic error in getFileContents for file ID {$fileId}: " . $e->getMessage());
            return response()->json(['error' => "An unexpected error has occurred on the server."], 500);
        }
    }

    /**
     * NEW FUNCTION: Gets temporary, pre-authenticated download URLs for multiple files from Box.
     */
    public function getDownloadUrls(Request $request)
    {
        $fileIds = $request->input('fileIds'); // Expecting an array of file IDs
        $accessToken = session('access_token');

        if (empty($fileIds) || !is_array($fileIds) || empty($accessToken)) {
            return response()->json(['error' => 'File IDs array and Access Token are required.'], 400);
        }

        try {
            $client = new Client(['verify' => false]);
            $header = [
                "Authorization" => "Bearer " . $accessToken,
                "Accept" => "application/json"
            ];

            $downloadUrls = [];

            // Box API for temporary download URLs does not support batching,
            // so we must loop. But this is very fast as it's just metadata.
            foreach ($fileIds as $fileId) {
                // The '?fields=download_url' is an optimization to only get the URL
                $requestURL = "https://api.box.com/2.0/files/" . $fileId . "?fields=download_url";

                try {
                    $response = $client->request('GET', $requestURL, ['headers' => $header]);
                    $fileInfo = json_decode($response->getBody()->getContents());

                    if (isset($fileInfo->download_url)) {
                        $downloadUrls[$fileId] = $fileInfo->download_url;
                    }
                } catch (\GuzzleHttp\Exception\ClientException $e) {
                    // Log the error for a specific file but continue for others
                    Log::error("Box API error getting download URL for file ID {$fileId}: " . $e->getResponse()->getBody()->getContents());
                }
            }

            return response()->json($downloadUrls);
        } catch (Exception $e) {
            return response()->json(['error' => "An error occurred while generating download URLs: " . $e->getMessage()], 500);
        }
    }
}


In the model
    /**
     * Summary of getDataByCenId
     * @param mixed $WSCenID
     * @param mixed $elementId
     */
    public function getCategoryNameByElementId($WSCenID, $elementIds)
    {
        $tableName = "doc_0201要素リスト";
        try {
            $placeholders = implode(',', array_fill(0, count($elementIds), '?'));
            $query = "SELECT  要素_ID, カテゴリー名,ファミリ名, タイプ_ID
                      FROM {$tableName}
                      WHERE WSCenID = ? AND 要素_ID IN ({$placeholders})";

            //Prepand WSCenId to the bindings array
            $bindings = array_merge([$WSCenID], $elementIds);
            $result = DB::connection('dldwh')->select($query, $bindings);

            $keyedResult = [];
            foreach ($result as $row) {
                $keyedResult[$row->要素_ID] = $row;
            }
            return response()->json($keyedResult);
        } catch (Exception $e) {
            return "An error has occurred: " . $e->getMessage();
        }
    }

<img width="1887" height="898" alt="image" src="https://github.com/user-attachments/assets/70770748-7537-4612-a33f-4e49bc39f268" />

