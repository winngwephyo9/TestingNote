 $dldwhModle = new DLDHWDataImportModel();
        return $dldwhModle->getCategoryNameByElementId($WSCenID, $elementIds);



in the model
 /**
     * Summary of getDataByCenId
     * @param mixed $WSCenID
     * @param mixed $elementId
     */
    public function getCategoryNameByElementId($WSCenID, $elementIds)
    {
        $tableName = "doc_0201要素リスト";
        try {
            $placeholders = implode(',', array_fill(0, count($elementIds), '?'));
            $query = "SELECT  要素_ID, カテゴリー名,ファミリ名, タイプ_ID
                      FROM {$tableName}
                      WHERE WSCenID = ? AND 要素_ID IN ({$placeholders})";

            //Prepand WSCenId to the bindings array
            $bindings = array_merge([$WSCenID], $elementIds);
            $result = DB::connection('dldwh')->select($query, $bindings);

            $keyedResult = [];
            foreach ($result as $row) {
                $keyedResult[$row->要素_ID] = $row;
            }
            return response()->json($keyedResult);
        } catch (Exception $e) {
            return "An error has occurred: " . $e->getMessage();
        }
    }
}
