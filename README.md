<img width="345" height="120" alt="image" src="https://github.com/user-attachments/assets/81ff9cda-a102-41e3-ab6e-62e41c216afd" />

/**
     * 【OBJ】フロントエンド用にデータを整形する
     */
    private function formatDataForFrontend($groupedFiles)
    {
        $objMtlPairs = [];
        foreach ($groupedFiles as $baseName => $files) {
            $objFile = $files->firstWhere('file_type', 'obj');
            if ($objFile && $objFile->file_path && Storage::disk('local')->exists($objFile->file_path)) {
                try {
                    $objContent = Storage::disk('local')->get($objFile->file_path);
                    $mtlContent = null;
                    $mtlFile = $files->firstWhere('file_type', 'mtl');
                    if ($mtlFile && $mtlFile->file_path && Storage::disk('local')->exists($mtlFile->file_path)) {
                        $mtlContent = Storage::disk('local')->get($mtlFile->file_path);
                    }
                    $objMtlPairs[] = [
                        'baseName' => $baseName,
                        'obj' => ['name' => $objFile->file_name, 'content' => $objContent],
                        'mtl' => $mtlFile ? ['name' => $mtlFile->file_name, 'content' => $mtlContent] : null
                    ];
                } catch (Exception $e) {
                    Log::error("Could not read file from storage: " . $e->getMessage());
                }
            }
        }
        return $objMtlPairs;
    }

    
objViewerBaraBara.js:322 Failed to render model: Error: 表示できるモデルデータがありません。
    at fetchAndRenderModel (objViewerBaraBara.js:246:19)
