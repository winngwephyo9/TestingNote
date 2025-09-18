in the job file
i moved db code into model
    private function saveJobLog($status, $message = null)
    {
        $dldwhModel = new DLDHWDataImportModel();
        $$dldwhModel->jobLogsUpdateOrInsert($this->projectFolderId, $status, $message);
    }

  public function jobLogsUpdateOrInsert($projectFolderId, $status, $message)
    {
        DB::connection('dldwh')->table('job_logs')->updateOrInsert(
            ['job_identifier' => $projectFolderId], // このキーで検索
            [ // 以下の内容で更新または作成
                'status' => $status,
                'message' => $message,
                'created_at' => now(),
                'updated_at' => now()
            ]
        );
    }

error occur
[2025-09-18 19:13:03] local.ERROR: Undefined variable: [] {"exception":"[object] (ErrorException(code: 0): Undefined variable: [] at C:\\xampp\\htdocs\\CCC\\ccc\\app\\Jobs\\SyncBoxProject.php:57)
[stacktrace]
#0 C:\\xampp\\htdocs\\CCC\\ccc\\app\\Jobs\\SyncBoxProject.php(57): Illuminate\\Foundation\\Bootstrap\\HandleExceptions->handleError(8, 'Undefined varia...', 'C:\\\\xampp\\\\htdocs...', 57, Array)
#1 C:\\xampp\\htdocs\\CCC\\ccc\\app\\Jobs\\SyncBoxProject.php(96): App\\Jobs\\SyncBoxProject->saveJobLog('failed', 'Undefined varia...')
