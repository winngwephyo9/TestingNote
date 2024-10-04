
![image](https://github.com/user-attachments/assets/efe10b5f-c527-4048-9d43-8f5cc5a9724c)

# Bigquery

/**
 * BigQueryReportExportService Class
 */
public class BigQueryReportExportService extends AbstractCdataReportExportService {
  private static final Logger Log = LogManager.getLogger(BigQueryReportExportService.class);

  private JsonElement exportOptionJsonElement;
  private Integer differentialUpdateFlag;
  private List<ExportColumnInfo> exportColumnInfoList = new ArrayList<>();
  private List<String> primaryKeys = new ArrayList<>();
  private List<Integer> pkIndexList = new ArrayList<>();
  private List<String[]> exportRecords = new ArrayList<>();
  private List<String> resultMessage = new ArrayList<>();
  private String columnNamesStr;
  private String tempTableName;
  private boolean isFirstExport = false;

  /**
   * Change data types from datafile side and cdata side as data types supported
   * from BigQuery
   */
  private static final Map<String, String> EXPORTED_DATA_TYPE = new HashMap<>();
  static {
    EXPORTED_DATA_TYPE.put("string", "STRING");
    EXPORTED_DATA_TYPE.put("bigint", "NUMERIC");
    EXPORTED_DATA_TYPE.put("double", "NUMERIC");
    EXPORTED_DATA_TYPE.put("date", "DATE");
    EXPORTED_DATA_TYPE.put("datetime", "TIMESTAMP");
    EXPORTED_DATA_TYPE.put("boolean", "BOOLEAN");
    EXPORTED_DATA_TYPE.put("BIT", "BOOLEAN");
    EXPORTED_DATA_TYPE.put("VARCHAR", "STRING");
    EXPORTED_DATA_TYPE.put("DECIMAL", "NUMERIC");
  }

  /**
   * BigQueryReportExportService Constructor
   * 
   * @param pApiParameter
   * @param pExportApiParameter
   * @param pDatasourceJdbc
   * @param pDatasourceCon
   */
  public BigQueryReportExportService(ApiParameter pApiParameter, ExportApiParameter pExportApiParameter,
      AbstractJdbc pDatasourceJdbc, Connection pDatasourceCon) {
    super();
    apiParameter = pApiParameter;
    serviceType = (String) pApiParameter.getServiceType();
    targetTableName = modifyTableName(apiParameter, pExportApiParameter.tableName);
    tempTableName = modifyTableName(apiParameter, pExportApiParameter.tableName.concat("#TEMP"));
    isFirstExport = pExportApiParameter.isFirstExport;
    exportOptionJsonElement = new JsonParser().parse(String.valueOf(pExportApiParameter.options));

    // Get body parameters from exported json 
    // And, set primary keys for `PrimaryKeyIdentifiers` for differential updated process before connecting to driver
    getExportedRequestBodyParams();

    datasourceJdbc = DatasourceJdbcFactory.create(serviceType, apiParameter);
    datasourceCon = ConnectionFactory.getConnection(datasourceJdbc);

    if (pExportApiParameter.insertValues != null) {
      // modify exported records from List<List<String>> to List<String[]> format
      exportRecords = pExportApiParameter.insertValues.stream().map(arr -> arr.toArray(new String[0]))
          .collect(Collectors.toList());
    }
  }

  /**
   * Export reports data
   * 
   * @return List<String>
   * @throws SQLException
   */
  @Override
  public List<String> exportReports() throws SQLException {
    try {
      checkTableSchema();

      if (differentialUpdateFlag == 1 && exportRecords.size() > 0) {
        differentialUpdate();
      } else if (differentialUpdateFlag == 0) {
        allUpdate();
      }
    } catch (SQLException e) {
      Log.error(e);
      throw e;
    }

    return resultMessage;
  }

  /**
   * Get options' json object
   *
   * @return optionJsonObj
   */
  private JsonObject getOptionsJsonObject() {
    if (!exportOptionJsonElement.isJsonObject())
      throw new NoSuchElementException("Options value is not json object type.");

    JsonObject settingOptionsJsonObj = exportOptionJsonElement.getAsJsonObject().get("setting_options")
        .getAsJsonObject();

    if (!settingOptionsJsonObj.keySet().contains("modify_columns")
        || !settingOptionsJsonObj.keySet().contains("primary_keys")
        || !settingOptionsJsonObj.keySet().contains("differential_update"))
      throw new NoSuchElementException(
          "Exported parameters are not contained correctly in setting_options json object");

    return settingOptionsJsonObj;
  }

  /**
   * Get export columns information from option object
   */
  private void getExportedRequestBodyParams() {
    JsonObject optionJsonObj = this.getOptionsJsonObject();
    differentialUpdateFlag = optionJsonObj.get("differential_update").getAsInt();

    Gson gson = new Gson();
    // retrieve export columns from Option Json Obj
    exportColumnInfoList = gson.fromJson(optionJsonObj.get("modify_columns").getAsJsonArray(),
        new TypeToken<ArrayList<ExportColumnInfo>>() {
        }.getType());

    // modify data type of export columns
    exportColumnInfoList.replaceAll(
        column -> new ExportColumnInfo(column.getColumnLabel(), EXPORTED_DATA_TYPE.get(column.getColumnType())));

    // concatenation column names with `,` in one String
    columnNamesStr = exportColumnInfoList.stream().map(ExportColumnInfo::getColumnLabel)
        .map(column -> modifyName(column)).collect(Collectors.joining(","));

    if (differentialUpdateFlag == 1) {
      primaryKeys = gson.fromJson(optionJsonObj.get("primary_keys"), new TypeToken<List<String>>() {
      }.getType());
      // check if primary key exists
      if (primaryKeys.isEmpty())
        throw new SystemException("At least a primary key is required for differential update process");

      // check if primary key exists in export column list
      List<String> columnNameList = exportColumnInfoList.stream().map(ExportColumnInfo::getColumnLabel)
          .collect(Collectors.toList());
      if (!columnNameList.containsAll(primaryKeys))
        throw new SystemException("Contain invalid primary key");

      // concatenation primary keys with `,` in one String
      String primaryKeysStr = primaryKeys.stream().map(pkColumn -> modifyName(pkColumn)).collect(Collectors.joining(","));
      // get index list of primary key columns
      pkIndexList = primaryKeys.stream().map(pk -> columnNameList.indexOf(pk)).collect(Collectors.toList());
      // set primary keys to use in connection builder
      apiParameter.setPrimaryKeyIdentifiers(primaryKeysStr);
    }
  }

  /**
   * Modify table schema information If table name does not exist, create new
   * table. Else if table name exist and data type is different, delete old column
   * and create again
   * 
   * @throws SQLException
   */
  private void checkTableSchema() throws SQLException {
    Statement stmt = datasourceCon.createStatement();
    Map<String, String> existingTableSchemaInfos = getTableSchemaInfos();

    if (existingTableSchemaInfos.isEmpty()) {
      stmt.executeUpdate(datasourceJdbc.getCreatedTableQuery(exportColumnInfoList, targetTableName));
    } else {
      modifySchemaInformation(existingTableSchemaInfos);
    }
    // reset schema cache to schema updates immediately
    stmt.executeUpdate("RESET SCHEMA CACHE;");
  }

  /**
   * Export by updating and inserting differential records
   * 
   * @throws SQLException
   */
  private void differentialUpdate() throws SQLException {
    // get existing records
    List<String> existingRecords = getExistingRecords();
    List<String[]> insertedRecords = new ArrayList<>();
    List<String[]> updatedRecords = new ArrayList<>();

    // group export records as inserted records and updated records by comparing existing records
    for (String[] recordArr : exportRecords) {
      if (existingRecords.contains(filterKey(recordArr))) {
        updatedRecords.add(recordArr);
      } else {
        insertedRecords.add(recordArr);
      }
    }

    if (insertedRecords.size() > 0)
      // insert exported new records
      insertRecords(insertedRecords);

    if (updatedRecords.size() > 0)
      // update exported records that primary keys of records are existing
      updateRecords(updatedRecords);

    // check LastResultInfo's result
    checkLastUpdatedResult();
  }

  /**
   * Export by deleting and inserting all records
   * 
   * @throws SQLException
   */
  private void allUpdate() throws SQLException {
    // if first time export of exported data file, delete all records from table
    if (isFirstExport)
      deleteRecords();

    if (exportRecords.size() > 0) {
      // insert exported new records
      insertRecords(exportRecords);
      // check LastResultInfo's result
      checkLastUpdatedResult();
    }
  }

  /**
   * Get table's schema information
   * 
   * @return fieldList
   * @throws SQLException
   */
  private Map<String, String> getTableSchemaInfos() throws SQLException {
    ResultSet result = datasourceCon.createStatement()
        .executeQuery(datasourceJdbc.getTableSchemaQuery(targetTableName));

    Map<String, String> fieldList = new HashMap<String, String>();
    while (result.next()) {
      String dataType = result.getString("DataTypeName");
      if (!EXPORTED_DATA_TYPE.containsValue(dataType) && StringUtils.isNotEmpty(EXPORTED_DATA_TYPE.get(dataType))) {
        dataType = EXPORTED_DATA_TYPE.get(dataType);
      }
      fieldList.put(result.getString("ColumnName"), dataType);
    }
    return fieldList;
  }

  /**
   * Get existing records' key values
   * 
   * @return existingRecords
   * @throws SQLException
   */
  private List<String> getExistingRecords() throws SQLException {
    String primaryKeysStr = joinPrimaryKeyColumns();
    List<String> primaryKeyValues = exportRecords.stream().map(r -> filterKey(r)).collect(Collectors.toList());
    String existingRecordQuery = datasourceJdbc.getExistingRecordQuery(primaryKeyValues.size(), targetTableName,
        primaryKeysStr);
    PreparedStatement stmt = datasourceCon.prepareStatement(existingRecordQuery);
    for (int colIdx = 0; colIdx < primaryKeyValues.size(); colIdx++) {
      stmt.setString(colIdx + 1, primaryKeyValues.get(colIdx));
    }

    List<String> existingRecords = new ArrayList<String>();
    ResultSet rs = stmt.executeQuery();
    while (rs.next()) {
      for (int i = 1; i <= rs.getMetaData().getColumnCount(); i++) {
        existingRecords.add(rs.getString(i));
      }
    }
    return existingRecords;
  }

  /**
   * Insert records Add inserted record to temp table Insert to main table by
   * selecting from temp table
   * 
   * @param insertedRecords
   * @throws SQLException
   */
  private void insertRecords(List<String[]> insertedRecords) throws SQLException {
    Statement stmt = datasourceCon.createStatement();
    String insertTempQuery = datasourceJdbc.getInsertedTempQuery(columnNamesStr, tempTableName);
    List<String> insertValues = insertedRecords.stream()
        .map(r -> Arrays.asList(r).stream().map(v -> modifyExportedValue(v)).collect(Collectors.joining(",")))
        .collect(Collectors.toList());
    for (String value : insertValues) {
      stmt.addBatch(String.format(insertTempQuery, value));
    }
    stmt.executeBatch();

    int insertResultCount = stmt
        .executeUpdate(datasourceJdbc.getInsertedSelectQuery(columnNamesStr, targetTableName, tempTableName));
    resultMessage.add("insert count: " + insertResultCount);
  }

  /**
   * Update records Add inserted record to temp table Update to main table by
   * selecting from temp table
   * 
   * @param updatedRecords
   * @throws SQLException
   */
  private void updateRecords(List<String[]> updatedRecords) throws SQLException {
    Statement stmt = datasourceCon.createStatement();
    String insertTempQuery = datasourceJdbc.getInsertedTempQuery(columnNamesStr, tempTableName);
    LinkedHashMap<String, String[]> modifiedRecord = updatedRecords.stream()
        .collect(Collectors.toMap(r -> filterKey(r), Function.identity(), (r1, r2) -> r2, LinkedHashMap::new));
    List<String> insertValues = modifiedRecord.values().stream()
        .map(r -> Arrays.asList(r).stream().map(v -> modifyExportedValue(v)).collect(Collectors.joining(",")))
        .collect(Collectors.toList());
    for (String value : insertValues) {
      stmt.addBatch(String.format(insertTempQuery, value));
    }
    stmt.executeBatch();

    int updateResultCount = stmt
        .executeUpdate(datasourceJdbc.getUpdatedSelectQuery(columnNamesStr, targetTableName, tempTableName));
    resultMessage.add("update count: " + updateResultCount);
  }

  /**
   * Delete records
   * 
   * @throws SQLException
   */
  private void deleteRecords() throws SQLException {
    PreparedStatement deleteStmt = datasourceCon.prepareStatement(datasourceJdbc.getDeleteQuery(targetTableName));
    deleteStmt.execute();
  }

  /**
   * Check last updated result success or not
   * 
   * @throws SQLException
   */
  private void checkLastUpdatedResult() throws SQLException {
    ResultSet result = datasourceCon.createStatement().executeQuery("SELECT * FROM LastResultInfo#TEMP");
    while (result.next()) {
      if (!result.getBoolean("Success"))
        throw new SQLException(result.getString("ErrorDescription"));
    }
  }

  /**
   * Get difference column list
   * 
   * @param existingColumnList
   */
  private List<List<String>> getDifferenceColumns(Map<String, String> existingColumnList) {
    // column list to drop : existing columns that are same column name but different column type
    List<String> deletedColumnList = new ArrayList<>(); 
    // column list to create : new columns and existing columns that are same column name but different column type
    List<String> addedColumnList = new ArrayList<>();
    // last column to drop : only one column that is last column from all different columns (same column name but different column type)
    List<String> lastDeletedColumn = new ArrayList<>();
    // last column to create : only one column that is last column from all different columns (same column name but different column type)
    List<String> lastAddedColumn = new ArrayList<>();

    for (ExportColumnInfo exportColumnInfo : exportColumnInfoList) {
      String exportedColumnName = exportColumnInfo.getColumnLabel();
      String exportedColumnType = exportColumnInfo.getColumnType();
      String existingIgnoredCaseColumnLabel = existingColumnList.keySet().stream()
          .filter(existedColumnName -> existedColumnName.equalsIgnoreCase(exportedColumnName)).findAny().orElse(null);

      if (existingIgnoredCaseColumnLabel == null) {
        // add new column
        addedColumnList.add(exportedColumnName + " " + exportedColumnType);
      } else {
        String existingColumnType = existingColumnList.entrySet().stream()
            .filter(existedColumnInfo -> existedColumnInfo.getKey().equalsIgnoreCase(exportedColumnName))
            .map(filteredColumnInfo -> filteredColumnInfo.getValue()).collect(Collectors.joining());
        // if column is not same data type with existing, modify column by deleting and creating again
        if (!StringUtils.equals(existingColumnType, exportedColumnType)) {
          int remainColumnCount = existingColumnList.size() - deletedColumnList.size();
          if (remainColumnCount == 1) {
            lastDeletedColumn.add(exportedColumnName);
            lastAddedColumn.add(exportedColumnName + " " + exportedColumnType);
          } else {
            deletedColumnList.add(exportedColumnName);
            addedColumnList.add(exportedColumnName + " " + exportedColumnType);
          }
        }
      }
    }

    List<List<String>> modifyColumnInfoLists = new ArrayList<>();
    modifyColumnInfoLists.add(deletedColumnList);
    modifyColumnInfoLists.add(addedColumnList);
    modifyColumnInfoLists.add(lastDeletedColumn);
    modifyColumnInfoLists.add(lastAddedColumn);

    return modifyColumnInfoLists;
  }

  /**
   * Modify schema information
   * 
   * @param existingColumnList
   * @throws SQLException 
   */
  private void modifySchemaInformation(Map<String, String> existingColumnList) throws SQLException {
    // get all of deleted column list and added column list
    List<List<String>> modifyColumnInfoLists = getDifferenceColumns(existingColumnList);
    List<String> deletedColumnList = modifyColumnInfoLists.get(0);
    List<String> addedColumnList = modifyColumnInfoLists.get(1);
    List<String> lastDeletedColumn = modifyColumnInfoLists.get(2);
    List<String> lastAddedColumn = modifyColumnInfoLists.get(3);

    Statement stmt = datasourceCon.createStatement();

    if (!deletedColumnList.isEmpty())
      stmt.executeUpdate(datasourceJdbc.getDeletedColumnQuery(deletedColumnList, targetTableName));

    if (!addedColumnList.isEmpty())
      stmt.executeUpdate(datasourceJdbc.getAddedColumnQuery(addedColumnList, targetTableName));

    // delete and create again for last column of existing table
    if (!lastDeletedColumn.isEmpty() && !lastAddedColumn.isEmpty()) {
      if (existingColumnList.size() > 1) {
        // if total column count are more than 1 column, delete and create again columns
        stmt.executeUpdate(datasourceJdbc.getDeletedColumnQuery(lastDeletedColumn, targetTableName));
        stmt.executeUpdate(datasourceJdbc.getAddedColumnQuery(lastAddedColumn, targetTableName));
      } else {
        // if total column count is only 1 column
        String tempColumn = lastDeletedColumn.get(0).concat("_temp");
        // create a temp column
        stmt.executeUpdate(
            datasourceJdbc.getAddedColumnQuery(Arrays.asList(tempColumn.concat(" STRING")), targetTableName));
        // delete existing main column
        stmt.executeUpdate(datasourceJdbc.getDeletedColumnQuery(lastDeletedColumn, targetTableName));
        // create again main column
        stmt.executeUpdate(datasourceJdbc.getAddedColumnQuery(lastAddedColumn, targetTableName));
        // delete temp column
        stmt.executeUpdate(datasourceJdbc.getDeletedColumnQuery(Arrays.asList(tempColumn), targetTableName));
      }
      deleteRecords();
    }
  }

  /**
   * Join primary key columns as one string with `,` 
   * And, modify `TIMESTAMP` data type columns for BigQuery TIMESTAMP format
   * 
   * @return String
   */
  private String joinPrimaryKeyColumns() {
    // get columns with `TIMESTAMP` data type
    List<String> timestampColumns = exportColumnInfoList.stream()
        .filter(columnInfo -> StringUtils.equals(columnInfo.getColumnType(), "TIMESTAMP"))
        .map(ExportColumnInfo::getColumnLabel).collect(Collectors.toList());
    String formatTimestampSubStr = "FORMAT_TIMESTAMP('%Y-%m-%d %H:%M:%S', ";
    // concatenation primary keys with `,` in one String
    String primaryKeysStr = primaryKeys.stream()
        .map(pkColumn -> timestampColumns.contains(pkColumn)
            ? formatTimestampSubStr.concat(String.format("`%s`", pkColumn)).concat(")")
            : String.format("`%s`", pkColumn))
        .collect(Collectors.joining(","));
    return primaryKeysStr;
  }

  /**
   * Get filtered key value
   * 
   * @param record
   * @return String
   */
  private String filterKey(String[] record) {
    return pkIndexList.stream().map(index -> record[index]).collect(Collectors.joining(""));
  }

  /**
   * Modify name with backtick According to BigQuery Console
   * 
   * @param name
   * @return String
   */
  private String modifyName(String name) {
    return "`" + StringUtil.escapeByBackSlash(name) + "`";
  }

  /**
   * Modify exported value format with single quote as '{value}'
   * 
   * @param value
   * @return String
   */
  private String modifyExportedValue(String value) {
    value = value.replace("\\", "\\\\").replace("'", "\\'");
    return String.format("'%s'", value);
  }

  /**
   * Modify table name as `{project_id}.{dataset_id}.{table_name}`
   * 
   * @param apiParameter
   * @param tableName
   * @return String
   */
  private String modifyTableName(ApiParameter apiParameter, String targetTableName) {
    targetTableName = targetTableName.replace("\\", "\\\\").replace("`", "\\`");
    StringBuilder tableNameBuilder = new StringBuilder();
    tableNameBuilder.append(apiParameter.getProjectId())
                    .append(".")
                    .append(apiParameter.getDataSetId())
                    .append(".")
                    .append(targetTableName);
    return String.format("`%s`", tableNameBuilder.toString());
  }
}

=============================================


# GoogleSheet

/**
 * GoogleSheetsReportExportService Class
 */
public class GoogleSheetsReportExportService extends AbstractCdataReportExportService {
  private static final Logger logger = LogManager.getLogger(GoogleSheetsReportExportService.class);
  private boolean deleteFlag = false;
  private List<String[]> records = new ArrayList<String[]>();
  private final static int ALPHABET_COUNT = 26;

  public GoogleSheetsReportExportService(ApiParameter pApiParameter, ExportApiParameter pExportApiParameter,
      AbstractJdbc pDatasourceJdbc, Connection pDatasourceCon) {
    super();
    apiParameter = pApiParameter;
    serviceType = (String) pApiParameter.getServiceType();
    if (pExportApiParameter.insertValues != null) {
      exportValues = pExportApiParameter.insertValues.stream().map(arr -> arr.toArray(new String[0]))
          .toArray(String[][]::new);
      records = Arrays.stream(exportValues).collect(Collectors.toList());
      exportFields = generateColumnList();
    }
    targetTableName = pExportApiParameter.tableName;
    deleteFlag = pExportApiParameter.deleteFlag;
    datasourceJdbc = DatasourceJdbcFactory.create(serviceType, apiParameter);
    datasourceCon = ConnectionFactory.getConnection(datasourceJdbc);
  }

  /**
   * Export reports data
   * @return List<String>
   * @throws SQLException
   */
  @Override
  public List<String> exportReports() throws SQLException {
    long startTime = System.currentTimeMillis();
    StopWatch sw = StopWatch.create(startTime);

    List<String> resultMessage = new ArrayList<>();

    try {
      if (deleteFlag) {
        // bulk DELETE
        deleteRecords();
        logger.printf(Level.INFO, "bulk-delete end. elapsed time: %,dms", sw.lapTime());
      }

      if (!records.isEmpty()) {
        // bulk INSERT
        int resultCountSum = insertRecords(records);
        resultMessage.add("insert count: " + resultCountSum);
        logger.printf(Level.INFO, "bulk-insert end. elapsed time: %,dms", sw.lapTime());
      }

    } catch (SQLException e) {
      throw e;
    }

    return resultMessage;
  }

  /**
   * Delete row for existing records
   * @throws SQLException
   */
  private void deleteRecords() throws SQLException {
    String deleteQuery = datasourceJdbc.getDeleteQuery(targetTableName);
    try {
      PreparedStatement deleteStmt = datasourceCon.prepareStatement(deleteQuery);
      deleteStmt.execute();
    } catch (SQLException e) {
      String errorLogMsg = this.showLogForExportError("DELETE処理でエラーが発生しました。: ", targetTableName, deleteQuery, e.getMessage());
      throw new SQLException(errorLogMsg);
    }
  }

  /**
   * データ追加（INSERT）を行う
   * @param insertRecords 追加レコードのリスト
   * @return int 追加レコード件数
   * @throws SQLException
   */
  private int insertRecords(List<String[]> insertRecords) throws SQLException {
    int insertRecordsCount = 0;
    String insertQuery = datasourceJdbc.getInsertQuery(exportFields, targetTableName);
    try {
      PreparedStatement insPstmt = datasourceCon.prepareStatement(insertQuery, Statement.RETURN_GENERATED_KEYS);
      for (String[] row : insertRecords) {
        // INSERT時はIdをスキップするため、列indexは1から開始する
        for (int colIdx = 0; colIdx < row.length; colIdx++) {
          insPstmt.setString(colIdx + 1, row[colIdx]);
        }
        insPstmt.addBatch();
      }
      int[] resultCounts = insPstmt.executeBatch();
      insertRecordsCount = Arrays.stream(resultCounts).sum();
    } catch (SQLException e) {
      String errorLogMsg = this.showLogForExportError("INSERT処理でエラーが発生しました。: ", targetTableName, insertQuery, e.getMessage());
      throw new SQLException(errorLogMsg);
    }
    return insertRecordsCount;
  }

  /**
   * Generate default columns array
   * @return String[]
   */
  private String[] generateColumnList() {
    int columnCount = this.getExportColumnCount();
    String[] columnNameArr = new String[columnCount];
    for (int i = 0; i < columnCount; i++) {
      columnNameArr[i] = getColumnName(i + 1);
    }
    return columnNameArr;
  }

  /**
   * Get column count to export
   * @return int
   */
  private int getExportColumnCount() {
    if (!records.isEmpty() && records.size() > 0) {
      return records.get(0).length;
    }
    return 0;
  }

  /**
   * Get generated default column name
   * @param columnNumber
   * @return String
   */
  private String getColumnName(int columnNumber) {
    StringBuilder columnName = new StringBuilder();
    while (columnNumber > 0) {
      int result = columnNumber % ALPHABET_COUNT;
      if (result == 0) {
        columnName.append("Z");
        columnNumber = (columnNumber / ALPHABET_COUNT) - 1;
      } else {
        columnName.append((char) ((result - 1) + 'A'));
        columnNumber = columnNumber / ALPHABET_COUNT;
      }
    }
    columnName.reverse();
    return columnName.toString();
  }

  /**
   * Show error log for insert and delete process
   * 
   * @param customMsg  String
   * @param reportName String
   * @param query      String
   * @param errorMsg   String
   * @return String
   */
  private String showLogForExportError(String customMsg, String reportName, String query, String errorMsg) {
    JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("report_name", reportName);
    jsonObject.addProperty("query", query);
    jsonObject.addProperty("error_message", errorMsg);
    String errorLogMsg = customMsg + LogUtil.replaceForJson(new Gson().toJson(jsonObject));
    logger.error(errorLogMsg);
    return errorLogMsg;
  }
}
