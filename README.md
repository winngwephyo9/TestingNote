 $filesToUpdate = [];
            foreach ($boxFiles as $boxFile) {
                $dbFile = $dbFiles->get($boxFile->id);
                
                if (!$dbFile) {
                    // DBにファイルが存在しない場合は、無条件で更新対象に追加
                    $filesToUpdate[] = $boxFile;
                    continue; // 次のループへ
                }

                // =================================================================
                //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
                //
                //  **【重要】マイクロ秒を切り捨ててから比較する**
                //
                $boxModifiedAt = Carbon::parse($boxFile->modified_at)->startOfSecond(); // マイクロ秒を.000000に丸める
                $dbModifiedAt = $dbFile->box_modified_at->startOfSecond();      // こちらも同様に丸める

                if ($dbModifiedAt->lt($boxModifiedAt)) {
                    // Boxの日時の方が新しい場合のみ更新対象とする
                    Log::info("File needs update [{$boxFile->name}]. DB: {$dbModifiedAt->toIso8601String()}, Box: {$boxModifiedAt->toIso8601String()}");
                    $filesToUpdate[] = $boxFile;
                }
