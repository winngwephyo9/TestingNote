public function organizeData($updatedDateWithoutSeconds, $folderItems, $access_token)
{
    $DLDWHModel = new DLDHWDataImportModel();
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

                    $boxDate = $this->getDateOrNameOfCsv($fileId, "date", $access_token);
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
                                Log::warning("Header and row count mismatch in file: {$fileName}");
                                continue;
                            }

                            $rowData = array_combine($header, $row);
                            $tableRow = [];
                            foreach ($allColumns as $column) {
                                $tableRow[$column] = $column === 'Updated_Date' ? $updatedDateWithoutSeconds : ($rowData[$column] ?? null);
                            }
                            $batchData[] = $tableRow;

                            if (count($batchData) >= 1000) { // バッチサイズを調整
                                $DLDWHModel->insertData($folderName, $batchData);
                                $batchData = [];
                            }
                        }

                        if (!empty($batchData)) {
                            $DLDWHModel->insertData($folderName, $batchData);
                        }

                        fclose($stream);
                        $DLDWHModel->updateCsvFileDate($fileName, Carbon::now());
                    }
                }
            }
        }
    }
    return []; // 処理結果を返さない
}
