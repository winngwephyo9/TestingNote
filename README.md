  /**
     * 【最終版】削除されたファイルをDBとストレージからクリーンアップする
     */
    private function cleanupDeletedFiles($projectFolderId, $boxFilesById)
    {
        $dbFileIds = ModelFileCache::where('project_box_id', $projectFolderId)->pluck('box_file_id');
        $boxFileIds = $boxFilesById->keys();
        $deletedIds = $dbFileIds->diff($boxFileIds)->all();

        if (!empty($deletedIds)) {
            Log::info("Deleting " . count($deletedIds) . " records from database...");
            ModelFileCache::whereIn('box_file_id', $deletedIds)->delete();
        }
    }
