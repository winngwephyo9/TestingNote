        $this->saveAccessTokenAfterLogin(session('access_token'), session('refresh_token'));
        SyncBoxProject::dispatch($projectFolderId);



    [2025-09-19 16:22:29] local.INFO: Job started for project: 341288887011  
[2025-09-19 16:22:30] local.ERROR: Box Sync Failed: Box access token is missing. {"exception":"[object] (Exception(code: 0): Box access token is missing. at C:\\xampp\\htdocs\\CCC\\ccc\\app\\Models\\DLDHWDataImportModel.php:5380)
[stacktrace]
#0 C:\\xampp\\htdocs\\CCC\\ccc\\app\\Jobs\\SyncBoxProject.php(71): App\\Models\\DLDHWDataImportModel->syncAndGetModelData('341288887011', NULL)
#1 C:\\xampp\\htdocs\\CCC\\ccc\\vendor\\laravel\\framework\\src\\Illuminate\\Container\\BoundMethod.php(36): App\\Jobs\\SyncBoxProject->handle()
