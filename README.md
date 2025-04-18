


/**
     * Execute the job.
     *![画像 (14)](https://github.com/user-attachments/assets/90df5b7b-8f57-4458-a3c9-591d07b23f82)
![画像 (15)](https://github.com/user-attachments/assets/197d95ba-9e65-44be-98e9-361adf2e0193)

     * @return void
     */
    public function handle()
    {
        $this->client = new \GuzzleHttp\Client(['verify' => false]);

        $dldwhImportController = new DLDWHDataImportController();
        $DLDWHModel = new DLDHWDataImportModel();
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
                $this->organizeAndProcessData($this->updatedDateWithoutSeconds, $csvFiles, $this->access_token, $this->updatedDate);

                Log::info('job finished uploadDataToDB');

                Log::info('start updateDMTable');
                $DLDWHModel = new DLDHWDataImportModel();
                // マテリアル容積
                $latestTimestamp = $DLDWHModel->update_DMVolume_MaterialVolume($DLDWHModel->getLatestUpdatedDate_document());
                $DLDWHModel->update_DMVolume_MaterialVolume_RC($latestTimestamp);
                $DLDWHModel->update_DMVolume_MaterialVolume_Ton($latestTimestamp);

                // ALC・ECP
                $latestTimestamp = $DLDWHModel->update_DMArea_0100WallArea($DLDWHModel->getLatestUpdatedDate_document());
                $DLDWHModel->update_DMArea_0101WallArea_ALC($latestTimestamp);
                $DLDWHModel->update_DMArea_0101WallArea_ECP($latestTimestamp);

                // 床仕上面積
                $DLDWHModel->update_DM_Area_Floor($DLDWHModel->getLatestUpdatedDate_document());
                // 壁仕上面積
                $DLDWHModel->update_DM_Area_Wall($DLDWHModel->getLatestUpdatedDate_document());
                // LGSのボード面積
                $DLDWHModel->update_DM_Area_Layer($DLDWHModel->getLatestUpdatedDate_document());

                Log::info('end updateDMTable');

                // 最終取込履歴の保存
                $DLDWHModel->saveUpdateHistory($this->loginUserName, $this->updatedDate);

                // ジョブ完了の保存
                $successMessage = [];
                $jsonResultList = $DLDWHModel->getDataImportLogs();
                foreach ($jsonResultList as $result) {
                    array_push($successMessage, json_decode($result->import_logs, true));
                }
                $finishJob = [
                    'fileNames' => $this->fileNameList,
                    'folderNames' => $this->folderNameList,
                    'successMessage' => $successMessage,
                ];
                $jsonFinishJob = json_encode($finishJob);
                // $DLDWHModel->saveFinishedJob($jsonFinishJob);
            } else {
                $finishJob = ["no_upload_file"];
                $jsonFinishJob = json_encode($finishJob);
                // $DLDWHModel->saveFinishedJob($jsonFinishJob);
            }
        } catch (\Exception $e) {
            Log::error('DLDWHJob failed: ' . $e->getMessage(), [
                'file' => $e->getFile(),
                'line' => $e->getLine(),
                'trace' => $e->getTraceAsString()
            ]);
            $this->fail($e);
        }
    }

    public function organizeAndProcessData($updatedDateWithoutSeconds, $folderItems, $access_token, $updatedDate)
{
    $DLDWHModel = new DLDHWDataImportModel();
    $dldwhImportController = new DLDWHDataImportController();
    $sortedTableNames = $DLDWHModel->showDependencyGraph();

    $normalizedFolderItems = [];
    foreach ($folderItems as $key => $value) {
        $normalizedFolderItems[strtolower($key)] = $value;
    }

    foreach ($sortedTableNames as $folderName) {
        if (!isset($normalizedFolderItems[strtolower($folderName)])) {
            Log::info("Organize Data Skip tables>>>>>" . $folderName);
            continue;
        }
        Log::info($folderName);
        $csvFiles = $normalizedFolderItems[$folderName];
        $tableExit = $DLDWHModel->checkTableExist($folderName);
        $allColumns = $DLDWHModel->getTableColumns($folderName);

        if ($tableExit) {
            foreach ($csvFiles as $files) {
                foreach ($files as $file) {
                    $fileName = $file['name'];
                    $fileId = $file['id'];
                    $isFileUpdated = false;

                    $boxDate = $dldwhImportController->getDateOrNameOfCsv($fileId, "date", $access_token);
                    $boxDateParsed = Carbon::parse($boxDate);
                    $csvFileDate = $DLDWHModel->getCsvFileDate($fileName);

                    if (!is_null($csvFileDate)) {
                        $lastModifiedAt = $csvFileDate->last_modified_at;
                        $fileDateParsed = Carbon::parse($lastModifiedAt);
                        if ($boxDateParsed > $fileDateParsed) {
                            $isFileUpdated = true;
                        }
                    }

                    if (is_null($csvFileDate) || $isFileUpdated) {
                        $requestURL = "https://api.box.com/2.0/files/" . $fileId . "/content";
                        $header = [
                            "Authorization" => "Bearer " . $access_token,
                            "Accept" => "application/json"
                        ];
                        $response = $this->client->request('GET', $requestURL, ['headers' => $header]);
                        $stream = $response->getBody()->detach();

                        $firstLine = fgets($stream);
                        if (substr($firstLine, 0, 3) === "\xEF\xBB\xBF") {
                            $firstLine = substr($firstLine, 3);
                        }

                        $encoding = mb_detect_encoding($firstLine, ['UTF-8', 'SJIS', 'EUC-JP', 'ISO-2022-JP']);
                        $firstLine = mb_convert_encoding($firstLine, 'UTF-8', $encoding);
                        $header = array_map('trim', explode(',', $firstLine));
                        $batchData = [];
                        while (($line = fgets($stream)) !== false) {
                            $line = mb_convert_encoding($line, 'UTF-8', $encoding);
                            $row = str_getcsv($line);

                            if (count($header) !== count($row)) {
                                // Log::warning("Header and row count mismatch in file: {$fileName}");
                                continue;
                            }

                            $rowData = array_combine($header, $row);
                            $tableRow = [];
                            foreach ($allColumns as $column) {
                                $tableRow[$column] = $column === 'Updated_Date' ? $updatedDateWithoutSeconds : ($rowData[$column] ?? null);
                            }
                            $batchData[] = $tableRow;

                            if (count($batchData) >= 1000) { // Adjust batch size as needed
                                $result = $dldwhImportController->uploadDataToDB($folderName, $allColumns, $batchData, $isFileUpdated);
                                $this->processResult($result, $folderName, $fileName, $DLDWHModel);
                                $batchData = [];
                            }
                        }

                        if (!empty($batchData)) {
                            $result = $dldwhImportController->uploadDataToDB($folderName, $allColumns, $batchData, $isFileUpdated);
                            $this->processResult($result, $folderName, $fileName, $DLDWHModel);
                        }

                        fclose($stream);
                        $DLDWHModel->saveCsvFileDate($fileName, $updatedDate, $isFileUpdated);

                        if(is_array($this->fileNameList) && is_array($this->folderNameList)){
                            array_push($this->fileNameList, $fileName);
                            array_push($this->folderNameList, $folderName);
                        } else {
                            $this->fileNameList = [$fileName];
                            $this->folderNameList = [$folderName];
                        }

                    }
                }
            }
            if ($folderName === $this->DOC_0201_ELEMENT_LISTS) {
                $latestUpdateDate = $DLDWHModel->getLatestUpdatedDate_element();
                $DLDWHModel->copyDataProfileList($this->DOC_0201_ELEMENT_LISTS, $this->DOC_0211_PROFILE_LISTS, $latestUpdateDate);
                $DLDWHModel->copyDataElementList($this->DOC_0201_ELEMENT_LISTS, $this->DOC_0201_ELEMENT_LEVEL_LISTS, $latestUpdateDate);
            }
        }
    }

    function processResult($result, $folderName, $fileName, $DLDWHModel)
    {
        $duplicates = [];
        $combinedKey = $folderName . '/' . $fileName;
        if (!isset($duplicates[$combinedKey])) {
            $duplicates[$combinedKey] = [];
        }
        if (isset($result['success']) && $result['success'] === 'duplicate') {
            $duplicates[$combinedKey][] = $result['duplicateRows'];
        } else {
            $duplicates[$combinedKey][] = "OK";
        }

        if (isset($duplicates)) {
            $jsonResult = json_encode($duplicates);
            $DLDWHModel->saveDataImportLogs($jsonResult);
        }
    }
