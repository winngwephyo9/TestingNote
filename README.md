/**
     * 【修正版】OBJビューア関連のテーブルを除外して、データを削除する
     */
    public function deleteData()
    {
        Log::info('Delete data process started...');

        // =================================================================
        //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
        //
        //  **【重要】削除処理から除外するテーブルのリストを定義**
        //
        $excludedTables = [
            'model_file_cache',       // OBJ/MTL/CSVのキャッシュ
            'box_tokens',             // Boxの認証トークン
            'job_logs',               // CSVインポートのジョブログ
            'job_logs_bara',          // OBJビューアの同期ジョブログ
            'jobs',                   // Laravelの標準ジョブテーブル
            'failed_jobs',            // Laravelの標準失敗ジョブテーブル
            'migrations',             // Laravelのマイグレーション管理テーブル
            'box_expiry_status',      // Boxのトークン有効期限ステータス
            // 他にも削除したくないテーブルがあれば、ここに追加
        ];
        //
        //  ▲▲▲-▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
        // =================================================================

        try {
            DB::connection('dldwh')->transaction(function () use ($excludedTables) {
                DB::connection('dldwh')->statement('SET FOREIGN_KEY_CHECKS=0;');

                // 'dldwh'接続のデータベース名を取得
                $databaseName = DB::connection('dldwh')->getDatabaseName();
                
                // テーブル情報を取得するクエリ
                $tables = DB::connection('dldwh')->select("SELECT table_name FROM information_schema.tables WHERE table_schema = '{$databaseName}'");

                foreach ($tables as $table) {
                    $tableName = $table->table_name;
                    
                    // 【重要】テーブル名が除外リストに含まれていないかチェック
                    if (!in_array($tableName, $excludedTables)) {
                        Log::info("Truncating table: {$tableName}");
                        DB::connection('dldwh')->table($tableName)->truncate();
                    } else {
                        Log::info("Skipping excluded table: {$tableName}");
                    }
                }

                DB::connection('dldwh')->statement('SET FOREIGN_KEY_CHECKS=1;');
            });
            
            Log::info('Delete data process completed successfully.');
            return response()->json(['status' => 'success']);

        } catch (\Exception $e) {
            Log::error('An error occurred during deleteData process: ' . $e->getMessage());
            // エラーが発生した場合も、外部キー制約は必ず元に戻す
            DB::connection('dldwh')->statement('SET FOREIGN_KEY_CHECKS=1;');
            return response()->json(['status' => 'error', 'message' => $e->getMessage()], 500);
        }
    }
