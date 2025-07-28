public function getFileContents(Request $request)
{
    $fileId = $request->input('fileId');
    $access_token = session('access_token'); // Simplified for clarity

    if (empty($fileId) || empty($access_token)) {
        return response('File ID or Access Token is missing.', 400);
    }
    
    try {
        $client = new Client(['verify' => false]);
        $requestURL = "https://api.box.com/2.0/files/" . $fileId . "/content";
        $header = [
            "Authorization" => "Bearer " . $access_token,
        ];

        $response = $client->request('GET', $requestURL, ['headers' => $header]);

        // --- THE FIX IS HERE ---
        // Get the raw body content of the file from the response
        $fileContent = $response->getBody()->getContents();

        // Return the raw content with an appropriate content type
        return response($fileContent, 200)->header('Content-Type', 'text/plain');
        // --- END OF FIX ---

    } catch (Exception $e) {
        // Log the actual error for debugging
        \Log::error("Box API file content fetch failed for file ID {$fileId}: " . $e->getMessage());
        return response('Failed to fetch file from Box.', 500);
    }
}


// This function now acts as a simple wrapper for your backend endpoint
async function fetchBoxFileContent(fileId) {
    // console.log("Fetching content for File ID:", fileId);
    try {
        const fileContent = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getFileContents", // Your new backend route
                data: { _token: CSRF_TOKEN, fileId: fileId },
                success: resolve, // On success, resolve the promise with the raw text data
                error: (jqXHR, textStatus, errorThrown) => {
                    // Reject with a meaningful error
                    reject(new Error(`AJAX error for file ${fileId}: ${textStatus} - ${errorThrown}`));
                }
            });
        });
        
        // The promise resolves with the file content directly.
        // We can add a simple check to see if we got something back.
        if (typeof fileContent !== 'string' || fileContent.length === 0) {
            throw new Error(`Received empty or invalid content for file ${fileId}`);
        }

        return fileContent; // Return the raw text content

    } catch (error) {
        console.error(`Failed inside fetchBoxFileContent for file ID ${fileId}:`, error);
        // Re-throw the error so Promise.all in the main loader function can catch it
        throw error; 
    }
}
