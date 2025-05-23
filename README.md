In the command file, call  ScraperEmailDataController of uploadToBox()
in the uploadToBox(){
 try {
            $request_url = 'https://upload.box.com/api/2.0/files/content';
            $access_token = session('access_token'); // Ensure this session variable is set
            if (!$access_token) {
                throw new \Exception('Box access token not found in session.');
            }

            $header = [
                "Authorization" => "Bearer " . $access_token,
            ];
            $multipart =  [
                [
                    'name'     => 'attributes',
                    'contents' => json_encode([
                        'name'      => $upload_file_name,
                        'parent'    => ['id' => $folder_id]
                    ])
                ],
                [
                    'name' => 'file',
                    'contents' => $upload_file,
                    'filename' => $upload_file_name,
                ]
            ];

            $client = new \GuzzleHttp\Client();
            $response = $client->request('POST', $request_url, [
                'headers'   => $header,
                'multipart' => $multipart
            ]);

            if ($response->getStatusCode() < 200 || $response->getStatusCode() >= 300) {
                throw new \Exception('Failed to upload file to Box. Status Code: ' . $response->getStatusCode() . ' Body: ' . $response->getBody());
            }
            log::info("File '{$upload_file_name}' uploaded to Box folder '{$folder_id}' successfully.");
}

I want to generate token once scheduler run this command
I am not use session token if session not has token,occur  Box access token not found in session error

reference this code that generate forge token, i want to generate box token
 function DoCopy($param_pj_id = null){
        $twoLeggedAuth = $this->GetTwoLeggedAuth();
        $twoleggedToken = $twoLeggedAuth->getAccessToken();
}

 function GetTwoLeggedAuth(){
        $conf = new \Autodesk\Auth\Configuration();//escape from current name space by using '/'
        $conf->getDefaultConfiguration()
        ->setClientId(env("FORGE_CLIENT_ID"))
        ->setClientSecret(env("FORGE_CLIENT_SECRET"));
        $twoLeggedAuth = new \Autodesk\Auth\OAuth2\TwoLeggedAuth();
        $scopes = array("code:all","data:read","data:write","data:create","bucket:read");
        $twoLeggedAuth->setScopes($scopes);

        $twoLeggedAuth->fetchToken();
        $token = $twoLeggedAuth->getAccessToken();

        return $twoLeggedAuth;
    }
