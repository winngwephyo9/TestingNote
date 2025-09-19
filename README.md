/**
     * 【修正版】Guzzle Poolに渡すリクエストの形式を修正
     */
    private function fetchMultipleBoxFileContentsConcurrently(array $files)
    {
        $downloadedContents = [];
        $tokenExpired = false;
        
        $client = new Client(['verify' => false]);
        $header = ["Authorization" => "Bearer " . session('access_token')];

        // =================================================================
        //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
        //
        //  **【重要】リクエストジェネレータの修正**
        //
        $requests = function ($files) use ($header) {
            foreach ($files as $file) {
                $url = "https://api.box.com/2.0/files/{$file['id']}/content";
                // GuzzleRequestオブジェクトそのものをyieldする
                yield $file['id'] => new GuzzleRequest('GET', $url, $header);
            }
        };
        //
        //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
        // =================================================================

        $pool = new Pool($client, $requests($files), [
            'concurrency' => 10,
            'fulfilled' => function ($response, $fileId) use (&$downloadedContents) {
                // fulfilledはキー（fileId）を受け取るので、マッピングは不要
                $downloadedContents[$fileId] = $response->getBody()->getContents();
            },
            'rejected' => function ($reason, $fileId) use (&$tokenExpired) {
                if ($reason instanceof ClientException && $reason->getResponse()->getStatusCode() == 401) {
                    $tokenExpired = true;
                } else {
                    Log::error("Failed to download file ID {$fileId}: " . $reason->getMessage());
                }
            }
        ]);

        $promise = $pool->promise();
        $promise->wait();

        if ($tokenExpired) {
            return ['status' => 'token_expired', 'contents' => []];
        }

        return ['status' => 'success', 'contents' => $downloadedContents];
    }
