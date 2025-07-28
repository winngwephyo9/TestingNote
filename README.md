use GuzzleHttp\Exception\ClientException; // <-- Add this at the top of your controller file

// ...

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

        $fileContent = $response->getBody()->getContents();
        return response($fileContent, 200)->header('Content-Type', 'text/plain');

    } catch (ClientException $e) {
        // --- THIS IS THE KEY IMPROVEMENT ---
        // This specifically catches HTTP errors from the API call (like 401 Unauthorized)
        $statusCode = $e->getResponse()->getStatusCode();
        $boxErrorBody = $e->getResponse()->getBody()->getContents();
        \Log::error("Box API ClientException for file ID {$fileId}: " . $boxErrorBody);
        
        // Return a specific error message to the frontend
        return response()->json([
            'error' => 'Box API Error', 
            'status' => $statusCode,
            'message' => 'Failed to fetch file from Box. The access token may have expired or permissions are insufficient.',
            'box_response' => json_decode($boxErrorBody) // Send the actual Box error back
        ], $statusCode); // Use the status code from Box

    } catch (Exception $e) {
        \Log::error("Generic error in getFileContents for file ID {$fileId}: " . $e->getMessage());
        return response()->json(['error' => "An unexpected error has occurred on the server."], 500);
    }
}
