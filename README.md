<?php
// ... (inside your FileController)

public function getFileContent(Request $request)
{
    $fileId = $request->input('fileId');
    $accessToken = $request->input('accessToken');

    if (empty($fileId) || empty($accessToken)) {
        return response()->json(['error' => 'File ID and Access Token are required.'], 400);
    }

    try {
        $client = new Client(['verify' => false]);
        $requestURL = "https://api.box.com/2.0/files/" . $fileId . "/content/";
        $header = [
            "Authorization" => "Bearer " . $accessToken,
        ];
        
        $response = $client->request('GET', $requestURL, ['headers' => $header]);
        
        // Return the raw file content directly
        return $response->getBody()->getContents();

    } catch (\GuzzleHttp\Exception\ClientException $e) {
        // Specifically catch Box API errors (like 429) and return a proper error response
        $statusCode = $e->getResponse()->getStatusCode();
        $body = $e->getResponse()->getBody()->getContents();
        Log::error("Box API ClientException for file ID {$fileId}: " . $body);
        return response()->json(['error' => 'Box API Error', 'details' => json_decode($body)], $statusCode);
    } catch (Exception $e) {
        Log::error("General Error fetching file ID {$fileId} from Box: " . $e->getMessage());
        return response()->json(['error' => "Failed to get file content from Box."], 500);
    }
}



// --- NEW AJAX Function for fetching a single file via your backend ---
function fetchFileContentViaServer(fileId) {
    return new Promise((resolve, reject) => {
        $.ajax({
            type: "post",
            url: window.URL_PREFIX + "/box/getFileContent", // Use the new route
            data: { _token: window.CSRF_TOKEN, fileId: fileId, accessToken: BOX_ACCESS_TOKEN },
            success: function(content) {
                resolve(content);
            },
            error: function(jqXHR, textStatus, errorThrown) {
                console.error(`AJAX error for file ID ${fileId}: ${textStatus} - ${errorThrown}`);
                // Pass along the error response from the server if available
                reject(new Error(`AJAX error for file ID ${fileId}: ${jqXHR.responseJSON ? JSON.stringify(jqXHR.responseJSON) : errorThrown}`));
            }
        });
    });
}


// --- REFACTORED Main Application Flow with Download Throttling ---
async function loadModel(projectFolderId, projectName) {
    // 0. Reset scene and state (this part remains the same)
    // ...

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list for ${projectName}...`;
        
        // 1. Fetch the list of OBJ/MTL file pairs
        const fileList = await new Promise((resolve, reject) => { /* ... (ajax call to /box/getObjList as before) ... */ });

        if (!fileList || !fileList.mtl || !fileList.objs || fileList.objs.length === 0) {
            throw new Error(`Incomplete file list for project "${projectName}".`);
        }

        const mtlFileInfo = fileList.mtl;
        const objFileInfoList = fileList.objs;

        // 2. Load the SINGLE MTL file ONCE
        if (loaderTextElement) loaderTextElement.textContent = `Loading Materials...`;
        const mtlContent = await fetchFileContentViaServer(mtlFileInfo.id); // Use new function
        const mtlLoader = new MTLLoader();
        const materialsCreator = mtlLoader.parse(mtlContent, '');
        materialsCreator.preload();

        // 3. **PERFORMANCE & RATE-LIMIT FIX**: Download all OBJ file contents in controlled parallel batches.
        if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (0/${objFileInfoList.length})...`;
        
        const CONCURRENT_DOWNLOADS = 8; // Keep this number low (e.g., 4-10)
        const allObjContents = [];
        let downloadedCount = 0;
        
        const downloadQueue = [...objFileInfoList];

        const downloadBatch = async () => {
            const currentBatchPromises = [];
            while (downloadQueue.length > 0 && currentBatchPromises.length < CONCURRENT_DOWNLOADS) {
                const objInfo = downloadQueue.shift();
                if (objInfo) {
                    const promise = fetchFileContentViaServer(objInfo.id).then(content => {
                        downloadedCount++;
                        if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (${downloadedCount}/${objFileInfoList.length})...`;
                        return { content: content, info: objInfo };
                    });
                    currentBatchPromises.push(promise);
                }
            }
            const results = await Promise.all(currentBatchPromises);
            allObjContents.push(...results);

            if (downloadQueue.length > 0) {
                await downloadBatch(); // Recursively call for the next batch
            }
        };

        await downloadBatch(); // Start the first batch

        // 4. Parse the header from the FIRST downloaded OBJ file
        // ... (This logic remains the same) ...
        
        // 5. Parse all OBJ geometries using the SAME material creator
        // ... (This logic remains the same) ...

        // ... (The rest of the function: combine, process, fetch categories, etc. remains the same) ...

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model: ${projectName}. Check console.`;
    }
}

// --- The rest of your script.js file remains the same ---
