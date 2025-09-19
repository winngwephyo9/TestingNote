   public function failed(Throwable $exception)
    {
        $this->saveJobLog('failed', $exception->getMessage());
        Log::error("Sync job has failed for project {$this->projectFolderId}: " . $exception->getMessage());
    }

[2025-09-19 12:29:27] local.INFO: Job started for project: 341288887011  
[2025-09-19 12:29:27] local.ERROR: Sync job has failed for project 341288887011: Trying to get property 'access_token' of non-object  

