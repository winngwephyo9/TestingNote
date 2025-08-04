

<img width="1512" height="804" alt="image" src="https://github.com/user-attachments/assets/8e535136-351a-40ba-90a8-0933f89c3536" />


034FB506718648D2

<?php
// ... (inside your FileController)

// This function can be kept, but we won't use it in the new flow
// public function getFileContent(Request $request) { ... }

/**
 * NEW FUNCTION: Gets temporary, pre-authenticated download URLs for multiple files from Box.
 */
public function getDownloadUrls(Request $request)
{
    $fileIds = $request->input('fileIds'); // Expecting an array of file IDs
    $accessToken = $request->input('accessToken');

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

// Add the new route
Route::post('/box/getDownloadUrls', [FileController::class, 'getDownloadUrls']);

// --- REFACTORED Main Application Flow with Direct Download from Box ---
async function loadModel(projectFolderId, projectName) {
    // 0. Reset scene and state (remains the same)
    // ...

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list for ${projectName}...`;
        
        // 1. Fetch the list of OBJ/MTL file IDs
        const fileList = await new Promise((resolve, reject) => { /* ... (ajax call to /box/getObjList) ... */ });
        if (!fileList || !fileList.mtl || !fileList.objs || fileList.objs.length === 0) {
            throw new Error(`Incomplete file list for project "${projectName}".`);
        }

        const mtlFileInfo = fileList.mtl;
        const objFileInfoList = fileList.objs;

        // 2. Get all file IDs to fetch
        const allFileIds = [mtlFileInfo.id, ...objFileInfoList.map(f => f.id)];

        // 3. Make a SINGLE call to your backend to get all temporary download URLs
        if (loaderTextElement) loaderTextElement.textContent = `Preparing secure downloads...`;
        const downloadUrlMap = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: window.URL_PREFIX + "/box/getDownloadUrls",
                data: { _token: window.CSRF_TOKEN, fileIds: allFileIds, accessToken: BOX_ACCESS_TOKEN },
                success: resolve,
                error: reject
            });
        });

        // 4. Load the SINGLE MTL file ONCE directly from Box
        if (loaderTextElement) loaderTextElement.textContent = `Loading Materials...`;
        const mtlUrl = downloadUrlMap[mtlFileInfo.id];
        if (!mtlUrl) throw new Error(`Could not get download URL for MTL file ${mtlFileInfo.name}`);
        
        const mtlLoader = new MTLLoader();
        const materialsCreator = await mtlLoader.loadAsync(mtlUrl);
        materialsCreator.preload();

        // 5. Download all OBJ file contents in controlled parallel batches, DIRECTLY from Box
        if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (0/${objFileInfoList.length})...`;
        const CONCURRENT_DOWNLOADS = 10;
        const allObjContents = [];
        let downloadedCount = 0;
        const downloadQueue = [...objFileInfoList];

        const downloadBatch = async () => {
            const currentBatchPromises = [];
            while (downloadQueue.length > 0 && currentBatchPromises.length < CONCURRENT_DOWNLOADS) {
                const objInfo = downloadQueue.shift();
                const objUrl = downloadUrlMap[objInfo.id];
                if (objInfo && objUrl) {
                    const promise = fetch(objUrl).then(res => {
                        if (!res.ok) throw new Error(`Failed to download ${objInfo.name} from Box: ${res.statusText}`);
                        return res.text();
                    }).then(content => {
                        downloadedCount++;
                        if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (${downloadedCount}/${objFileInfoList.length})...`;
                        return { content, info: objInfo };
                    });
                    currentBatchPromises.push(promise);
                }
            }
            const results = await Promise.all(currentBatchPromises);
            allObjContents.push(...results);
            if (downloadQueue.length > 0) await downloadBatch();
        };

        await downloadBatch();

        // ... (The rest of the function: parse header, parse geometries, combine, process, fetch categories, etc., remains the same) ...

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model: ${projectName}. Check console.`;
    }
}


