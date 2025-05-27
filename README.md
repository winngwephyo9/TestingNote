if (strpos($contentType, 'text/csv') !== false || strpos($contentType, 'application/octet-stream') !== false) {

                $csvData = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'shift_jis');

                $fileName = $filePrefix . date('Ymd') . '.csv';



I want to store this csv data into box using power automade that schedule two times one month

Can you tell me details step
