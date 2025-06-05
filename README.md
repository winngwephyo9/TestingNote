   private function handleCsvDownload($response, $filePath, string $filePrefix): void
    {
        if ($response->getStatusCode() === 200) {
            $contentType = $response->getHeaderLine('Content-Type');
            if (strpos($contentType, 'text/csv') !== false || strpos($contentType, 'application/octet-stream') !== false) {
                // $csvData = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'shift_jis');
                $rawCsvData = (string) $response->getBody();

                // Convert from CP932 to UTF-8
                // $csvData = mb_convert_encoding($rawCsvData, 'UTF-8', 'CP932');
                $csvData = mb_convert_encoding($rawCsvData, 'UTF-8', 'Shift_JIS');

                // --- ADD THIS LINE TO PREPEND THE UTF-8 BOM ---
                // The UTF-8 BOM is represented by these three bytes: EF BB BF
                $csvDataWithBom = "\xEF\xBB\xBF" . $csvData;
                $fileName = $filePrefix . date('Ymd') . '.csv';
                $filePath = $filePath . "\\" . $fileName;
                file_put_contents($filePath, $csvDataWithBom);
                dump("{$fileName} downloaded and uploaded to Box successfully.");
            } else {
                throw new \Exception("Expected CSV data in response, but got: " . $contentType);
            }
        } else {
            throw new \Exception("Error submitting search on ADDR102.aspx: " . $response->getStatusCode());
        }
    }

![image](https://github.com/user-attachments/assets/fff32f8a-c8e7-44db-8ae3-3f25e6f05b2d)


