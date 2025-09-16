 $filesToUpdate = [];
            foreach ($boxFiles as $boxFile) {
                $dbFile = $dbFiles->get($boxFile->id);

                // =================================================================
                //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
                //
                //  **【重要】Carbonを使ってタイムゾーンを考慮した正確な比較を行う**
                //
                $boxModifiedAt = Carbon::parse($boxFile->modified_at); // Boxの日時をCarbonオブジェクトに変換

                if (!$dbFile || $dbFile->box_modified_at->lt($boxModifiedAt)) {
                    // DBにない、またはBoxの日時の方が新しい場合のみ更新対象とする
                    // lt() は "less than" (より小さい) を意味する
                    if ($dbFile) {
                        Log::info("File needs update [{$boxFile->name}]. DB: {$dbFile->box_modified_at}, Box: {$boxModifiedAt}");
                    }
                    $filesToUpdate[] = $boxFile;
                }
