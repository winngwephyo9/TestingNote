 public function getDataByCenId($WSCenID, $elementId)
    {
        $tableName = "doc_0201要素リスト";
        try {
            $query = "SELECT カテゴリー名, ファミリ名, タイプ_ID
                      FROM {$tableName}
                      WHERE WSCenID = :WSCenID AND 要素_ID = :elementId";

            $result = DB::connection('dldwh')->select($query, [
                'WSCenID' => $WSCenID,
                'elementId' => $elementId
            ]);

            return json_decode(json_encode($result), true);
        } catch (Exception $e) {
            return "An error has occurred: " . $e->getMessage();
        }
    }

![image](https://github.com/user-attachments/assets/30150b37-11d4-41c3-9b00-f69ad7a6abca)
