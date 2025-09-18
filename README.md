In the controller
 //Boxからデータを取り込む
            $DLDWHModel = new DLDHWDataImportModel();
            $DLDWHModel->deleteFinshedJob();
            $DLDWHModel->deleteDataImportLogs();
            $DLDWHModel->deleteFailedJobs();
            $DLDWHModel->deleteBoxExpiryStatus();
            DLDWHJob::dispatch(
                $this->parentFolderId,
                $updatedDateWithoutSeconds,
                $updatedDate,
                $loginUserName,
                $this->access_token,
                $this->refresh_token,
                $this->box_login_time,
                $this->url_path
            );

            $finishJob = [
                'isfinishJob' => false,
            ];
            return response()->json($finishJob);
        } else {
            $finishJob = [
                'token' => "no_token",
            ];
            return response()->json($finishJob);
        }
In the Job
 public function handle()
 {
 $this->client = new \GuzzleHttp\Client();
 $this->client = new \GuzzleHttp\Client(['verify' => false]);
 $DLDWHModel = new DLDHWDataImportModel();
 $dldwhImportController = new DLDWHDataImportController();

 try {
 Log::info('start LoadDataFromBox ');
 $csvFiles = $dldwhImportController->LoadDataFromBox(
 $this->parentFolderId,
 $this->updatedDateWithoutSeconds,
 $this->updatedDate,
 $this->access_token
 );

 if (!empty($csvFiles)) {
 Log::info('start organizeData ' . count($csvFiles));
 $latestUpdateDate = $DLDWHModel->getLatestUpdatedDate_document();
 $latestDate_KoSuu = $DLDWHModel->getLatestDateForKoSuu();
 $this->organizeAndProcessData($csvFiles);

 Log::info('job finished uploadDataToDB');

 // ジョブ完了の保存
 $successMessage = [];
 $jsonResultList = $DLDWHModel->getDataImportLogs();
 foreach ($jsonResultList as $result) {
 array_push($successMessage, json_decode($result->import_logs, true)); // trueを追加して配列としてデコード
 }
 $finishJob = [
 'fileNames' => $this->fileNameList,
 'folderNames' => $this->folderNameList,
 'successMessage' => $successMessage,
 ];
 $jsonFinishJob = json_encode($finishJob);

 $DLDWHModel->saveFinishedJob($jsonFinishJob);
 } else {
 //取り込むファイルがない
 $finishJob = ["no_upload_file"];
 $jsonFinishJob = json_encode($finishJob);
 $DLDWHModel->saveFinishedJob($jsonFinishJob);
 }
 } catch (Exception $e) {
 Log::error('DLDWHJob failed: ' . $e->getMessage(), [
 'file' => $e->getFile(),
 'line' => $e->getLine(),
 'trace' => $e->getTraceAsString()
 ]);
 $this->fail($e);
 }
 }
In the model
    /**
     * saveFinishedJob
     *
     * 全データ取り込んだ後、取り込んだ結果を記録する
     * 
     * @param  mixed $jsonFinishJob
     * @return string
     */
    public function saveFinishedJob($jsonFinishJob)
    {
        $query = "INSERT INTO job_logs (job_id, logs) VALUES (?, ?)";
        $result = DB::connection('dldwh')->insert($query, [1, $jsonFinishJob]);
        return $result;
    }

    /**
     * deleteFinshedJob
     *
     * 取り込んだ結果を削除する
     * @return void
     * 
     */
    public function deleteFinshedJob()
    {
        $deleted = DB::connection('dldwh')
            ->table('job_logs')
            ->delete();
    }

    /**
     * getFinishedJob
     *
     * 取り込んだ結果を取得する
     * @return mixed
     * 
     */
    public function getFinishedJob()
    {
        try {
            $result = DB::connection('dldwh')->table('job_logs')->first();
            return $result;
        } catch (Exception $e) {
            Log::error('Error fetching finished job: ' . $e->getMessage());
            return null;
        }
    }

    /**
     * getFailedJobs
     *
     * 失敗したジョブデータを取得する
     * @return mixed
     * 
     */
    public function getFailedJobs()
    {
        try {
            $result = DB::connection('dldwh')->table('failed_jobs')->first();
            return $result;
        } catch (Exception $e) {
            Log::error('Error fetching failed jobs: ' . $e->getMessage());
            return null;
        }
    }

    /**
     * deleteFailedJobs
     *
     * 失敗したジョブデータを削除する
     * @return void
     * 
     */
    public function deleteFailedJobs()
    {
        $deleted = DB::connection('dldwh')
            ->table('failed_jobs')
            ->delete();
    }

    /**
     * deleteBoxExpiryStatus
     *
     * deleteBoxExpiryStatusを削除する
     * @return void
     * 
     */
    public function deleteBoxExpiryStatus()
    {
        $deleted = DB::connection('dldwh')
            ->table('box_expiry_status')
            ->delete();
    }
