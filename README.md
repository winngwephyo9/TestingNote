    public function handle()
    {
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
    $jobData = $this->organizeData($this->updatedDateWithoutSeconds, $csvFiles, $this->access_token);

    Log::info('start job ' . count($jobData));
    $latestUpdateDate = $DLDWHModel->getLatestUpdatedDate_document();

    $this->doJob($jobData, $this->updatedDate);
    Log::info('job finished uploadDataToDB');

    Log::info('start updateDMTable');
    $DLDWHModel = new DLDHWDataImportModel();
    // マテリアル容積
    $latestTimestamp = $DLDWHModel->update_DMVolume_MaterialVolume($latestUpdateDate);
    $result_rc = $DLDWHModel->update_DMVolume_MaterialVolume_RC($latestTimestamp);
    $result_t = $DLDWHModel->update_DMVolume_MaterialVolume_Ton($latestTimestamp);

    // ALC・ECP
    $latestTimestamp = $DLDWHModel->update_DMArea_0100WallArea($latestUpdateDate);
    $result_alcALC = $DLDWHModel->update_DMArea_0101WallArea_ALC($latestTimestamp);
    $result_alcECP = $DLDWHModel->update_DMArea_0101WallArea_ECP($latestTimestamp);

    // 床仕上面積
    $DLDWHModel->update_DM_Area_Floor($latestUpdateDate);
    // 壁仕上面積
    $DLDWHModel->update_DM_Area_Wall($latestUpdateDate);
    // LGSのボード面積
    $DLDWHModel->update_DM_Area_Layer($latestUpdateDate);

    Log::info('end updateDMTable');

    // //最終取込履歴の保存
    $DLDWHModel->saveUpdateHistory($this->loginUserName, $this->updatedDate);

    //ジョーブ完了の保存
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
    $finishJob = [
    "no_upload_file"
    ];
    $jsonFinishJob = json_encode($finishJob);
    $DLDWHModel->saveFinishedJob($jsonFinishJob);
    }
    } catch (\Exception $e) { // すべての例外をキャッチ
    Log::error('DLDWHJob failed: ' . $e->getMessage(), [
    'file' => $e->getFile(),
    'line' => $e->getLine(),
    'trace' => $e->getTraceAsString()
    ]);
    $this->fail($e); // ジョブを失敗として記録
    }
    }

    public function organizeData($updatedDateWithoutSeconds, $folderItems, $access_token)
    {
    $DLDWHModel = new DLDHWDataImportModel();
    $jobData = [];
    // Get the sorted table names
    $sortedTableNames = $DLDWHModel->showDependencyGraph();

    // Normalize the case of folder names in $folderItems
    $normalizedFolderItems = [];
    foreach ($folderItems as $key => $value) {
    $normalizedFolderItems[strtolower($key)] = $value;
    }

    foreach ($sortedTableNames as $folderName) {
    // foreach ($folderItems as $folderName => $csvFiles) {
    if (!isset($normalizedFolderItems[strtolower($folderName)])) {
    log::info("Organize Data Skip tables>>>>>" . $folderName);
    continue; // Skip if the folder name is not in the folder items
    }
    log::info($folderName);
    $csvFiles = $normalizedFolderItems[$folderName];
    $tableExit = $DLDWHModel->checkTableExist($folderName);
    $allColumns = $DLDWHModel->getTableColumns($folderName);
    $collection = collect();

    if ($tableExit) {
    foreach ($csvFiles as $files) {
    foreach ($files as $file) {
    $fileName = $file['name'];
    $fileId = $file['id'];
    $isFileUpdated = false;

    $boxDate = $this->getDateOrNameOfCsv($fileId, "date", $access_token); //当csvのboxからの日付を取得
    $boxDateParsed = Carbon::parse($boxDate);
    $csvFileDate = $DLDWHModel->getCsvFileDate($fileName); //当csvのデータベースからの日付を取得

    //Box日付とデータベースの日付を比較し、Boxの日付が大きければファイルが更新されているとして処理する
    if (!is_null($csvFileDate)) {
    $lastModifiedAt = $csvFileDate->last_modified_at;
    $fileDateParsed = Carbon::parse($lastModifiedAt);
    if ($boxDateParsed > $fileDateParsed) {
    $isFileUpdated = true;
    }
    }

    //当csvがデータベースに保存されたことがないもしくは当csvが更新された場合、データを取得する
    if (is_null($csvFileDate) || $isFileUpdated) {
    $requestURL = "https://api.box.com/2.0/files/" . $fileId . "/content";
    $header = [
    "Authorization" => "Bearer " . $access_token,
    "Accept" => "application/json"
    ];
    $response = $this->client->request('GET', $requestURL, ['headers' => $header]);
    $files = $response->getBody()->getContents();

    if (substr($files, 0, 3) === "\xEF\xBB\xBF") {
    $files = substr($files, 3); // BOMを削除
    }

    // ファイルのエンコーディングを確認
    $encoding = mb_detect_encoding($files, ['UTF-8', 'SJIS', 'EUC-JP', 'ISO-2022-JP']);

    // エンコーディングをUTF-8に変換
    $files = mb_convert_encoding($files, 'UTF-8', $encoding);

    $lines = explode("\n", $files);
    $headerLine = array_shift($lines);
    $header = array_map('trim', explode(',', $headerLine));
    $tableData = [];
    log::info("Line Count >>>>>>>" . count($lines));

    foreach ($lines as $line) {
    if (empty(trim($line))) {
    continue; // 空行をスキップ
    }
    $row = explode(',', $line);
    log::info("Row Count >>>>>>>" . count($row));
    $rowData = array_combine($header, $row);
    $tableRow = [];
    foreach ($allColumns as $column) {
    $tableRow[$column] = $column === 'Updated_Date' ? $updatedDateWithoutSeconds : ($rowData[$column] ?? null);
    }
    $tableData[] = $tableRow;
    }

    $collection->push([
    'folderName' => $folderName,
    'tableData' => $tableData,
    'columns' => $allColumns,
    'isFileUpdated' => $isFileUpdated,
    'fileName' => $fileName,

    ]);
    }
    }
    }
    }
    array_push($jobData, $collection->toArray());
    }
    return $jobData;
    }

    public function doJob($csvFiles, $updatedDate)
    {
    $DLDWHModel = new DLDHWDataImportModel();
    $dldwhImportController = new DLDWHDataImportController();
    $dmVolumeTbl = [];
    $fileNameList = [];
    $folderNameList = [];
    $batchSize = 1000;
    $isUpdateDMTable = false;
    if (!empty($csvFiles)) {
    foreach ($csvFiles as $data) {
    foreach ($data as $item) {
    $allColumns = $item['columns'];
    $folderName = $item['folderName'];
    $tableData = $item['tableData'];
    $isFileUpdated = $item['isFileUpdated'];
    $fileName = $item['fileName'];
    $successMessage = [];
    $duplicates = [];
    array_push($fileNameList, $fileName);
    array_push($folderNameList, $folderName);

    //5000行ずつジョブへ渡して、データベースに挿入する
    for ($i = 0; $i < count($tableData); $i +=$batchSize) {
        $batch=array_slice($tableData, $i, $batchSize);
        $result=$dldwhImportController->uploadDataToDB($folderName, $allColumns, $batch, $isFileUpdated);

        array_push($successMessage, $result);
        }
        foreach ($successMessage as $sm) {
        $combinedKey = $folderName . '/' . $fileName; // Combine folderName and fileName
        if (!isset($duplicates[$combinedKey])) {
        $duplicates[$combinedKey] = [];
        }
        if (isset($sm['success']) && $sm['success'] === 'duplicate') {
        $duplicates[$combinedKey][] = $sm['duplicateRows'];
        } else {
        $duplicates[$combinedKey][] = "OK";
        }
        }

        if (isset($duplicates)) {
        $jsonResult = json_encode($duplicates);
        $DLDWHModel->saveDataImportLogs($jsonResult);
        }

        $DLDWHModel->saveCsvFileDate($fileName, $updatedDate, $isFileUpdated);
        // array_push($dmVolumeTbl, values: true);
        }
        log::info("Inserted Table Name>>>>>>>>>>>>>>>>>>>>>");
        log::info($folderName,);

        // Check if folderName matches DOC_0201_ELEMENT_LISTS and copy data to 0201_copy_element
        if ($folderName === $this->DOC_0201_ELEMENT_LISTS) {
        $latestUpdateDate = $DLDWHModel->getLatestUpdatedDate_element();
        $DLDWHModel->copyDataProfileList($this->DOC_0201_ELEMENT_LISTS, $this->DOC_0211_PROFILE_LISTS, $latestUpdateDate);
        $DLDWHModel->copyDataElementList($this->DOC_0201_ELEMENT_LISTS, $this->DOC_0201_ELEMENT_LEVEL_LISTS, $latestUpdateDate);
        }
        }
        $this->fileNameList = $fileNameList;
        $this->folderNameList = $folderNameList;
        Log::info('end upload csv files');
        }
        }
