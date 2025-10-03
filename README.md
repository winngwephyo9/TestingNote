/**
     * 【最終修正版】DBキャッシュからモデルデータを取得し、CSVとOBJペアに分離する
     */
    public function getCachedModelData($projectFolderId)
    {
        // =================================================================
        //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
        //
        //  **【重要】CSVと、その他のファイル（OBJ/MTL）を別々に取得する**
        //

        // 1. CSVファイルを検索・取得
        $csvFile = ModelFileCache::where('project_box_id', $projectFolderId)
                                 ->where('file_type', 'csv')
                                 ->first();

        // 2. OBJとMTLファイルだけを取得
        $objMtlFiles = ModelFileCache::where('project_box_id', $projectFolderId)
                                     ->whereIn('file_type', ['obj', 'mtl'])
                                     ->get();

        if ($objMtlFiles->isEmpty()) {
            return ['error' => 'No cached model files found.'];
        }

        // 3. OBJ/MTLファイルをbase_nameでグループ化してペアにする
        $objMtlPairs = [];
        $groupedFiles = $objMtlFiles->groupBy('base_name');

        foreach ($groupedFiles as $baseName => $files) {
            $objFile = $files->firstWhere('file_type', 'obj');
            $mtlFile = $files->firstWhere('file_type', 'mtl');

            // OBJファイルが存在する場合のみペアとして成立
            if ($objFile) {
                $objMtlPairs[] = [
                    'baseName' => $baseName,
                    'obj' => ['name' => $objFile->file_name, 'content' => $objFile->content],
                    'mtl' => $mtlFile ? ['name' => $mtlFile->file_name, 'content' => $mtlFile->content] : null
                ];
            }
        }
        
        //
        //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
        // =================================================================

        // 4. フロントエンドに返すための最終的なデータ構造を組み立てる
        return [
            'csv_data' => $csvFile ? ['name' => $csvFile->file_name, 'content' => $csvFile->content] : null,
            'obj_mtl_pairs' => $objMtlPairs
        ];
    }"Property [file_type] does not exist on this collection instance."

