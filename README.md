### Excel table
I want to create table filter that like excel table filter when click filter icon show popup box that below image
with html,css, and js can use require library
![image](https://github.com/user-attachments/assets/dea3106e-6902-4f60-8d33-8557f41440da)



###
$(document).ready(function () {
    slectAllCompanyName();
    LoadAllPartnerCompany(); // Ensure this is called when the page loads
});

export async function LoadAllPartnerCompany() {
    try {
        const response = await fetch(url_prefix + "/partnerMgmt/getAllPartnerCompany", {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-TOKEN': CSRF_TOKEN
            },
            body: JSON.stringify({ _token: CSRF_TOKEN })
        });

        if (!response.ok) {
            throw new Error('Network response was not ok');
        }

        const data = await response.json(); // Assuming the response is JSON

        if (data != null && data !== '') {
            ShowPartnerCompanyList(data);
        }
    } catch (err) {
        alert("データロードに失敗しました。\n管理者に問い合わせてください。");
    }
}

function ShowPartnerCompanyList(data) {
    $.each(data, function (tname, tdata) {
        var hasData = tdata.model_data && tdata.model_data.length > 0; // Check if company has data

        var row = "<tr data-id='" + tdata['company_id'] + "'><td class='company-cell'>" +
            "<input type='checkbox' class='company-checkbox' data-id='" + tdata['company_id'] + "' data-name='" + tdata['company_name'] + "'" + 
            (hasData ? " checked" : "") + // Pre-check the checkbox if the company has data
            "></td><td>" + tdata['company_name'] + "</td></tr>";

        $("#companyNameList tbody").append(row);

        // If the company has data, preselect and load the info
        if (hasData) {
            getPartnerCompanyInfo(tdata['company_id'], tdata['company_name'], true); // Pass true for selection
            selectedCompaniesData[tdata['company_id']] = tdata.model_data; // Store the data
            $("input[data-id='" + tdata['company_id'] + "']").closest('td').addClass("selected"); // Add selected class
        }
    });

    // Add change event to each company checkbox
    $(".company-checkbox").change(function () {
        var companyName = $(this).data("name");
        var companyId = $(this).data("id");

        if (this.checked) {
            getPartnerCompanyInfo(companyId, companyName, true); // Pass true for selection
            $(this).closest('td').addClass("selected");
        } else {
            updateTotalModelCount(selectedCompaniesData[companyId], false); // Use stored data for deselection
            removeCompanyInfo(companyId);
            delete selectedCompaniesData[companyId]; // Remove stored data
            $(this).closest('td').removeClass("selected");
        }
    });

    // Add search functionality
    $("#searchBox").on("keyup", function () {
        var value = $(this).val().toLowerCase();
        $("#companyNameList tbody tr").filter(function () {
            $(this).toggle($(this).text().toLowerCase().indexOf(value) > -1);
        });
    });
}

// Add event listener to the "Select All" checkbox
function slectAllCompanyName() {
    $("#selectAllCheckbox").change(function () {
        if (this.checked) {
            $(".company-checkbox").each(function () {
                if (!this.checked) {
                    this.checked = true;
                    $(this).trigger('change');
                }
            });
        } else {
            $(".company-checkbox").each(function () {
                if (this.checked) {
                    this.checked = false;
                    $(this).trigger('change');
                }
            });
        }
    });
}

function getPartnerCompanyInfo(id, company_name, isSelected) {
    ShowLoading1();
    var company_id = id ? id : $("#companyName").val();
    if (company_id == 0) {
        alert("会社名を選択してください。");
        return;
    } else {
        $.ajax({
            url: url_prefix + "/partnerMgmt/getModelCount",
            type: 'post',
            data: { _token: CSRF_TOKEN, company_id: company_id },
            success: function (data) {
                if (data == 'no company') {
                    alert("Error");
                } else {
                    if (isSelected) {
                        showCompanyInfo(data, company_name, company_id);
                        selectedCompaniesData[company_id] = data; // Store data for deselection
                    }
                    updateTotalModelCount(data, isSelected); // Pass isSelected flag
                }
                checkAllRequestsCompleted(); // Check if all requests are completed
            },
            error: function (err) {
                console.log(err);
                checkAllRequestsCompleted(); // Ensure to check even on error
            }
        });
    }
}

function showCompanyInfo(data, name, id) {
    const tableBody = document.querySelector('#companyInfoTable tbody');
    const groupedData = {};
    let totalCreate = 0;
    let totalCreateModify = 0;
    let totalModify = 0;
    let totalCost = 0;

    // Group data by model_type
    data.forEach(item => {
        if (!groupedData[item.model_type]) {
            groupedData[item.model_type] = { 作成: 0, 作成・修正: 0, 修正: 0 };
        }
        groupedData[item.model_type][item.model_value] = item.model_count;
        totalCost = item.total_cost;
    });

    const row1 = document.createElement('tr');
    row1.setAttribute('data-id', id);
    row1.classList.add('company-info-row');
    const r1c1 = document.createElement('td');
    r1c1.setAttribute("rowSpan", "3");
    r1c1.textContent = name;
    row1.appendChild(r1c1);

    const r1c2 = document.createElement('td');
    r1c2.classList.add('create');
    r1c2.textContent = "作成";
    row1.appendChild(r1c2);

    const row2 = document.createElement('tr');
    row2.setAttribute('data-id', id);
    row2.classList.add('company-info-row');
    const r2c2 = document.createElement('td');
    r2c2.classList.add('createModify');
    r2c2.textContent = "作成・修正";
    row2.appendChild(r2c2);

    const row3 = document.createElement('tr');
    row3.setAttribute('data-id', id);
    row3.classList.add('company-info-row');
    const r3c2 = document.createElement('td');
    r3c2.classList.add('modify');
    r3c2.textContent = "修正";
    row3.appendChild(r3c2);

    if (data.length === 0) {
        for (var col = 0; col < 12; col++) {
            const r1c3 = document.createElement('td');
            r1c3.textContent = "";
            row1.append(r1c3);

            const r2c3 = document.createElement('td');
            r2c3.textContent = "";
            row2.append(r2c3);

            const r3c3 = document.createElement('td');
            r3c3.textContent = "";
            row3.append(r3c3);
        }
    } else {
        Object.keys(groupedData).forEach(model_type => {
            const r1c3 = document.createElement('td');
            const createCount = groupedData[model_type]["作成"];
            r1c3.textContent = createCount ? createCount : "";
            row1.appendChild(r1c3);
            if (createCount) totalCreate += createCount;

            const r2c3 = document.createElement('td');
            const createModifyCount = groupedData[model_type]["作成・修正"];
            r2c3.textContent = createModifyCount ? createModifyCount : "";
            row2.appendChild(r2c3);
            if (createModifyCount) totalCreateModify += createModifyCount;

            const r3c3 = document.createElement('td');
            const modifyCount = groupedData[model_type]["修正"];
            r3c3.textContent = modifyCount ? modifyCount : "";
            row3.appendChild(r3c3);
            if (modifyCount) totalModify += modifyCount;
        });

        const totalCreateCell = document.createElement('td');
        totalCreateCell.textContent = totalCreate || "";
        row1.appendChild(totalCreateCell);

        const totalCreateModifyCell = document.createElement('td');
        totalCreateModifyCell.textContent = totalCreateModify || "";
        row2.appendChild(totalCreateModifyCell);

        const totalModifyCell = document.createElement('td');
        totalModifyCell.textContent = totalModify || "";
        row3.appendChild(totalModifyCell);
    }

    const totalCostRow = document.createElement('td');
    totalCostRow.setAttribute("rowSpan", "3");
    totalCostRow.textContent = totalCost || "";
    row1.appendChild(totalCostRow);

    tableBody.appendChild(row1);
    tableBody.appendChild(row2);
    tableBody.appendChild(row3);
}

function removeCompanyInfo(company_id) {
    const rows = document.querySelectorAll('#companyInfoTable tbody tr');
    rows.forEach(row => {
        if (row.getAttribute('data-id') == company_id) {
            row.remove();
        }
    });
}

function updateTotalModelCount(data, isSelected) {
    let totalCreate = 0;
    let totalCreateModify = 0;
    let totalModify = 0;
    let totalCost = 0;

    data.forEach(item => {
        if (item.model_value === '作成') totalCreate += item.model_count;
        if (item.model_value === '作成・修正') totalCreateModify += item.model_count;
        if (item.model_value === '修正') totalModify += item.model_count;
        totalCost = item.total_cost;
    });

    if (isSelected) {
        totalCreateModel += totalCreate;
        totalCreateModifyModel += totalCreateModify;
        totalModifyModel += totalModify;
        totalModelCost += totalCost;
    } else {
        totalCreateModel -= totalCreate;
        totalCreateModifyModel -= totalCreateModify;
        totalModifyModel -= totalModify;
        totalModelCost -= totalCost;
    }

    $("#totalCreate").text(totalCreateModel || "");
    $("#totalCreateModify").text(totalCreateModifyModel || "");
    $("#totalModify").text(totalModifyModel || "");
    $("#totalCost").text(totalModelCost || "");
}



#tab3
$(document).ready(function () {
    slectAllCompanyName();
});

export async function LoadAllPartnerCompany() {
    try {
        const response = await fetch(url_prefix + "/partnerMgmt/getAllPartnerCompany", {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-TOKEN': CSRF_TOKEN
            },
            body: JSON.stringify({ _token: CSRF_TOKEN })
        });

        if (!response.ok) {
            throw new Error('Network response was not ok');
        }

        const data = await response.json(); // Assuming the response is JSON

        if (data != null && data !== '') {
            ShowPartnerCompanyList(data);
        }
    } catch (err) {
        alert("データロードに失敗しました。\n管理者に問い合わせてください。");
    }
}

function ShowPartnerCompanyList(data) {
    $.each(data, function (tname, tdata) {
        var row = "<tr data-id='" + tdata['company_id'] + "'><td class='company-cell'><input type='checkbox' class='company-checkbox' data-id='" + tdata['company_id'] + "' data-name='" + tdata['company_name'] + "'></td><td>" + tdata['company_name'] + "</td></tr>";
        $("#companyNameList tbody").append(row);
    });
    // $.each(data, function (tname, tdata) {
    //     var row = "<tr data-id='" + tdata['company_id'] + "'><td class='company-cell'><input type='checkbox' class='company-checkbox' data-id='" + tdata['company_id'] + "' data-name='" + tdata['company_name'] + "'>" + tdata['company_name'] + "</td></tr>";
    //     $("#companyNameList tbody").append(row);
    // });

    // Add click event to each company checkbox
    $(".company-checkbox").change(function () {
        var companyName = $(this).data("name");
        var companyId = $(this).data("id");
        if (this.checked) {
            // selectedCompanies.push(companyId);
            getPartnerCompanyInfo(companyId, companyName, true); // Pass true for selection
            $(this).closest('td').addClass("selected");
        } else {
            updateTotalModelCount(selectedCompaniesData[companyId], false); // Use stored data for deselection
            removeCompanyInfo(companyId);
            delete selectedCompaniesData[companyId]; // Remove stored data
            $(this).closest('td').removeClass("selected");
        }
    });

    // Add search functionality
    $("#searchBox").on("keyup", function () {
        var value = $(this).val().toLowerCase();
        $("#companyNameList tbody tr").filter(function () {
            $(this).toggle($(this).text().toLowerCase().indexOf(value) > -1);
        });
    });
}


// Add event listener to the "Select All" checkbox
function slectAllCompanyName() {
    $("#selectAllCheckbox").change(function () {
        if (this.checked) {
            $(".company-checkbox").each(function () {
                if (!this.checked) {
                    this.checked = true;
                    $(this).trigger('change');
                }
            });
        } else {
            $(".company-checkbox").each(function () {
                if (this.checked) {
                    this.checked = false;
                    $(this).trigger('change');
                }
            });
        }
    });
}

function getPartnerCompanyInfo(id, company_name, isSelected) {
    ShowLoading1();
    var company_id = id ? id : $("#companyName").val();
    if (company_id == 0) {
        alert("会社名を選択してください。");
        return;
    } else {
        $.ajax({
            url: url_prefix + "/partnerMgmt/getModelCount",
            type: 'post',
            data: { _token: CSRF_TOKEN, company_id: company_id },
            success: function (data) {
                if (data == 'no company') {
                    alert("Error");
                } else {
                    if (isSelected) {
                        showCompanyInfo(data, company_name, company_id);
                        selectedCompaniesData[company_id] = data; // Store data for deselection
                    }
                    updateTotalModelCount(data, isSelected); // Pass isSelected flag
                }
                // HideLoading();
                checkAllRequestsCompleted(); // Check if all requests are completed
            },
            error: function (err) {
                console.log(err);
                checkAllRequestsCompleted(); // Ensure to check even on error
            }
        });
    }
}

function showCompanyInfo(data, name, id) {
    const tableBody = document.querySelector('#companyInfoTable tbody');
    const groupedData = {};
    let totalCreate = 0;
    let totalCreateModify = 0;
    let totalModify = 0;
    let totalCost = 0;

    // Group data by model_type
    data.forEach(item => {
        if (!groupedData[item.model_type]) {
            groupedData[item.model_type] = { 作成: 0, 作成・修正: 0, 修正: 0 };
        }
        groupedData[item.model_type][item.model_value] = item.model_count;
        totalCost = item.total_cost;
    });

    const row1 = document.createElement('tr');
    row1.setAttribute('data-id', id);
    row1.classList.add('company-info-row');
    const r1c1 = document.createElement('td');
    r1c1.setAttribute("rowSpan", "3");
    r1c1.textContent = name;
    row1.appendChild(r1c1);

    const r1c2 = document.createElement('td');
    r1c2.classList.add('create');
    r1c2.textContent = "作成";
    row1.appendChild(r1c2);

    const row2 = document.createElement('tr');
    row2.setAttribute('data-id', id);
    row2.classList.add('company-info-row');
    const r2c2 = document.createElement('td');
    r2c2.classList.add('createModify');
    r2c2.textContent = "作成・修正";
    row2.appendChild(r2c2);

    const row3 = document.createElement('tr');
    row3.setAttribute('data-id', id);
    row3.classList.add('company-info-row');
    const r3c2 = document.createElement('td');
    r3c2.classList.add('modify');
    r3c2.textContent = "修正";
    row3.appendChild(r3c2);

    if (data.length === 0) {
        for (var col = 0; col < 12; col++) {
            const r1c3 = document.createElement('td');
            r1c3.textContent = "";
            row1.append(r1c3);

            const r2c3 = document.createElement('td');
            r2c3.textContent = "";
            row2.append(r2c3);

            const r3c3 = document.createElement('td');
            r3c3.textContent = "";
            row3.append(r3c3);
        }
    } else {
        Object.keys(groupedData).forEach(model_type => {
            const r1c3 = document.createElement('td');
            const createCount = groupedData[model_type]["作成"];
            r1c3.textContent = createCount ? createCount : "";
            row1.appendChild(r1c3);
            if (createCount) totalCreate += createCount;

            const r2c3 = document.createElement('td');
            const createModifyCount = groupedData[model_type]["作成・修正"];
            r2c3.textContent = createModifyCount ? createModifyCount : "";
            row2.appendChild(r2c3);
            if (createModifyCount) totalCreateModify += createModifyCount;

            const r3c3 = document.createElement('td');
            const modifyCount = groupedData[model_type]["修正"];
            r3c3.textContent = modifyCount ? modifyCount : "";
            row3.appendChild(r3c3);
            if (modifyCount) totalModify += modifyCount;
        });

        const totalCreateCell = document.createElement('td');
        totalCreateCell.textContent = totalCreate || "";
        row1.appendChild(totalCreateCell);

        const totalCreateModifyCell = document.createElement('td');
        totalCreateModifyCell.textContent = totalCreateModify || "";
        row2.appendChild(totalCreateModifyCell);

        const totalModifyCell = document.createElement('td');
        totalModifyCell.textContent = totalModify || "";
        row3.appendChild(totalModifyCell);
    }

    const totalCostRow = document.createElement('td');
    totalCostRow.setAttribute("rowSpan", "3");
    totalCostRow.textContent = totalCost || "";
    row1.appendChild(totalCostRow);

    tableBody.appendChild(row1);
    tableBody.appendChild(row2);
    tableBody.appendChild(row3);
}

function removeCompanyInfo(company_id) {
    const rows = document.querySelectorAll('#companyInfoTable tbody tr');
    rows.forEach(row => {
        const aa = row.getAttribute('data-id');
        if (aa === String(company_id)) {
            row.remove();
        }
    });
}

function updateTotalModelCount(data, isSelected) {
    if (isSelected && data.length > 0) {
        totalAllCost += Number(data[0].total_cost);
    } else if (data.length > 0) {
        totalAllCost -= Number(data[0].total_cost);
    }
    data.forEach(item => {
        console.log("Each data = " + item.total_cost);

        if (isSelected) {
            totalModelCounts[item.model_type] += item.model_count;
        } else {
            totalModelCounts[item.model_type] -= item.model_count;
        }
    });

    const totalTable = document.querySelector('#total-table tbody');
    totalTable.innerHTML = '';
    const row1 = document.createElement('tr');
    const r1c1 = document.createElement('td');
    r1c1.textContent = "合計";
    row1.appendChild(r1c1);
    Object.keys(totalModelCounts).forEach(modelType => {
        const cell = document.createElement('td');
        cell.textContent = totalModelCounts[modelType];
        row1.appendChild(cell);
    });

    const totalModelCountCell = document.createElement('td');
    totalModelCountCell.textContent = Object.values(totalModelCounts).reduce((a, b) => a + b, 0);
    row1.appendChild(totalModelCountCell);

    const totalCostCell = document.createElement('td');
    totalCostCell.textContent = totalAllCost;
    row1.appendChild(totalCostCell);
    totalTable.appendChild(row1);
}

function ShowLoading1() {
    $(".loading").removeClass("hide");
    $(".loading").addClass("show");
    totalRequests++;
}

function HideLoading1() {
    $(".loading").removeClass("show");
    $(".loading").addClass("hide");
}

function checkAllRequestsCompleted() {
    completedRequests++;
    if (completedRequests >= totalRequests) {
        HideLoading1();
        totalRequests = 0; // Reset counters
        completedRequests = 0;
    }
}

# ascending and descending
![image](https://github.com/user-attachments/assets/efe10b5f-c527-4048-9d43-8f5cc5a9724c)
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Sortable Table</title>
    <style>
        body {
    font-family: Arial, sans-serif;
}

table {
    width: 100%;
    border-collapse: collapse;
}

th, td {
    padding: 10px;
    text-align: left;
    border: 1px solid #000;
    background-color: #333;
    color: #fff;
    cursor: pointer;
}

th:hover {
    background-color: #555;
}

th::after {
    content: " ▼"; /* Default arrow (descending) */
}

th[data-order="asc"]::after {
    content: " ▲"; /* Ascending arrow */
}

    </style>
</head>
<body>
    <table id="sortable-table">
        <thead>
            <tr>
                <th data-column="projectName" data-order="desc">プロジェクト名称</th>
                <th data-column="pjCode" data-order="desc">PJコード</th>
                <th data-column="branchName" data-order="desc">支店名</th>
                <th data-column="reporterName" data-order="desc">報告者名</th>
                <th data-column="organizationName" data-order="desc">投稿者組織名</th>
            </tr>
        </thead>
        <tbody>
            <!-- Data will be injected here from API -->
        </tbody>
    </table>

    <script>
        // Mock API call to get table data
const fetchData = async () => {
    return [
        { projectName: "プロジェクトA", pjCode: "PJ001", branchName: "東京", reporterName: "田中", organizationName: "本社" },
        { projectName: "プロジェクトB", pjCode: "PJ002", branchName: "大阪", reporterName: "佐藤", organizationName: "支社" },
        { projectName: "プロジェクトC", pjCode: "PJ003", branchName: "名古屋", reporterName: "鈴木", organizationName: "地方" }
    ];
};

// Sort the table by the column clicked
const sortTable = (column, order) => {
    const table = document.querySelector('#sortable-table tbody');
    const rows = Array.from(table.rows);

    const sortedRows = rows.sort((rowA, rowB) => {
        const cellA = rowA.querySelector(`td[data-column="${column}"]`).innerText.toLowerCase();
        const cellB = rowB.querySelector(`td[data-column="${column}"]`).innerText.toLowerCase();

        if (cellA < cellB) return order === 'asc' ? -1 : 1;
        if (cellA > cellB) return order === 'asc' ? 1 : -1;
        return 0;
    });

    // Reorder the rows in the table
    table.append(...sortedRows);
};

// Populate table with data from API
const populateTable = async () => {
    const data = await fetchData();
    const tableBody = document.querySelector('#sortable-table tbody');

    data.forEach(item => {
        const row = document.createElement('tr');

        Object.keys(item).forEach(key => {
            const cell = document.createElement('td');
            cell.innerText = item[key];
            cell.setAttribute('data-column', key); // Add this for sorting purpose
            row.appendChild(cell);
        });

        tableBody.appendChild(row);
    });
};

// Event listener for sorting
document.querySelectorAll('th').forEach(header => {
    header.addEventListener('click', () => {
        const column = header.getAttribute('data-column');
        const currentOrder = header.getAttribute('data-order');
        const newOrder = currentOrder === 'desc' ? 'asc' : 'desc';

        // Update the order attribute
        header.setAttribute('data-order', newOrder);

        // Sort the table
        sortTable(column, newOrder);
    });
});

// Call the function to populate the table on page load
populateTable();

    </script>
</body>
</html>



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
