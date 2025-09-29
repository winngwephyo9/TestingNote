 foreach ($updateChunks as $chunkOfFiles) {
                // ストリーミングでダウンロードし、成功したファイルのメタデータのみを受け取る
                $result = $this->fetchAndStreamBoxFilesConcurrently($chunkOfFiles, $projectFolderId, $projectName, $accessToken, $refreshToken, $urlPath);
                $downloadedFilesMetadata = $result['metadata'];
                
                // 次のループで使うために、リフレッシュされた可能性のあるトークンを更新
                $accessToken = $result['new_access_token'];
                $refreshToken = $result['new_refresh_token'];

                if (!empty($downloadedFilesMetadata)) {
                    // DBへのUpsertはメタデータだけなので非常に高速
                    ModelFileCache::upsert(
                        $downloadedFilesMetadata, 
                        ['box_file_id'], 
                        ['project_name', 'file_name', 'base_name', 'file_type', 'file_path', 'box_modified_at', 'updated_at']
                    );
                }
            }
