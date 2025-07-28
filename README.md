async function fetchBoxFileContent(fileId) {
    console.log("Field Id of fetchBoxFileContent()", fileId);
    try {
        const response = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getFileContents",
                data: { _token: CSRF_TOKEN, fileId: fileId },
                success: resolve,
                error: reject
            });
        });
        if (!response.ok) throw new Error(`Failed to fetch file ${fileId} from Box: ${response.statusText}`);
        return await response.text();
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "Error fetching project list from Box.";
    }
}

in the controller
  public function getFileContents(Request $request)
    {
        $fileId = $request->input('fileId');

        if (session()->has('access_token')) {
            $access_token = session('access_token');
            if ($access_token == "") {
                return "no_token";
            }
            if (session()->has('authority_id')) {
                $login = new LoginModel();
                $result = $login->GetBoxAuthority(session('authority_id'));
                $client = new Client(['verify' => false]);
                $requestURL = "https://api.box.com/2.0/files/" . $fileId . "/content";
                $header = [
                    "Authorization" => "Bearer " . $access_token,
                    "Accept" => "application/json"
                ];

                $response = $client->request('GET', $requestURL, ['headers' => $header]);
                return $response;
            }
        } else {
            return "no_token";
        }
    }
