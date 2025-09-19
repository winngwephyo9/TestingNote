    /**
     * 【最終版】Boxからプロジェクト名（フォルダ名）を取得する
     */
    private function fetchProjectName($projectFolderId, Client $client)
    {
        $header = ["Authorization" => "Bearer " . session('access_token')];
        $folderInfoUrl = "https://api.box.com/2.0/folders/{$projectFolderId}?fields=name";
        $folderInfoResponse = $client->get($folderInfoUrl, ['headers' => $header]);
        return json_decode($folderInfoResponse->getBody()->getContents())->name;
    }
