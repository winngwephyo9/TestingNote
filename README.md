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

                        $boxDate = $this->getDateOrNameOfCsv($fileId, "date", $access_token);      //当csvのboxからの日付を取得
                        $boxDateParsed = Carbon::parse($boxDate);
                        $csvFileDate = $DLDWHModel->getCsvFileDate($fileName);                //当csvのデータベースからの日付を取得

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
            array_push($jobData,  $collection->toArray());
        }
        return $jobData;
    }
ERROR: Allowed memory size of 536870912 bytes exhausted (tried to allocate 20480 bytes) {"exception":"[object] (Symfony\\Component\\ErrorHandler\\Error\\FatalError(code: 0): Allowed memory size of 536870912 bytes exhausted (tried to allocate 20480 bytes) at $row = explode(',', $line);

Can you modify code to solve error and modify code to more effectively storage
