  // 【重要】ジョブ開始時にDBから最新のトークン情報を取得
            $dldwhModel = new DLDHWDataImportModel();
            $latestTokens = $dldwhModel->getLatestBoxTokens();
            if (!$latestTokens) {
                throw new Exception("No valid tokens found in the database.");
            }
