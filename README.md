    private function saveJobLog($status, $message = null)
    {
        $dldwhModel = new DLDHWDataImportModel();
        // 【修正済み】余分な$を削除
        $dldwhModel->jobLogsUpdateOrInsert($this->projectFolderId, $status, $message);
    }



       /**
     * job_logsテーブルに結果を保存/更新する
     */
    public function jobLogsUpdateOrInsert($projectFolderId, $status, $message)
    {
        DB::connection('dldwh')->table('job_logs')->updateOrInsert(
            ['job_identifier' => $projectFolderId],
            ['status' => $status, 'message' => $message, 'updated_at' => now()]
        );
    }
