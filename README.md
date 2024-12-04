<details open>
    <summary><strong>会社ID</strong></summary>
    <div class="custom-select-container">
        <input type="text" id="companyIdSearch" onkeyup="filterDropdown('companyId')" placeholder="検索" class="form-control search-box">
        <ul id="companyIdDropdown" class="dropdown-list"></ul>
    </div>
</details>

<details open>
    <summary><strong>協定書締結日</strong></summary>
    <div class="custom-select-container">
        <input type="text" id="agreementDateSearch" onkeyup="filterDropdown('agreementDate')" placeholder="検索" class="form-control search-box">
        <ul id="agreementDateDropdown" class="dropdown-list"></ul>
    </div>
</details>

<details open>
    <summary><strong>担当支店</strong></summary>
    <div class="custom-select-container">
        <input type="text" id="branchSearch" onkeyup="filterDropdown('branch')" placeholder="検索" class="form-control search-box">
        <ul id="branchDropdown" class="dropdown-list"></ul>
    </div>
</details>

<details open>
    <summary><strong>担当部門</strong></summary>
    <div class="custom-select-container">
        <input type="text" id="departmentSearch" onkeyup="filterDropdown('department')" placeholder="検索" class="form-control search-box">
        <ul id="departmentDropdown" class="dropdown-list"></ul>
    </div>
</details>

<details open>
    <summary><strong>会社名</strong></summary>
    <div class="custom-select-container">
        <input type="text" id="companyNameSearch" onkeyup="filterDropdown('companyName')" placeholder="検索" class="form-control search-box">
        <ul id="companyNameDropdown" class="dropdown-list"></ul>
    </div>
</details>
.custom-select-container {
    position: relative;
    width: 100%;
}

.search-box {
    margin-bottom: 5px;
    padding: 5px;
    width: 100%;
    box-sizing: border-box;
}

.dropdown-list {
    list-style-type: none;
    padding: 0;
    margin: 0;
    max-height: 200px;
    overflow-y: auto;
    border: 1px solid #ccc;
    background: #fff;
}

.dropdown-list li {
    padding: 5px;
    display: flex;
    align-items: center;
    cursor: pointer;
}

.dropdown-list li:hover {
    background-color: #f0f0f0;
}

function populateFilters() {
    populateDropdown('companyId', 'company_id');
    populateDropdown('agreementDate', 'sign_date');
    populateDropdown('branch', 'tantou_shiten');
    populateDropdown('department', 'tantou_bumon');
    populateDropdown('companyName', 'company_name');
}

function populateDropdown(dropdownId, dataKey) {
    const dropdown = document.getElementById(`${dropdownId}Dropdown`);
    const values = [...new Set(filterDataArray.map(item => item[dataKey]))];

    // Clear existing items
    dropdown.innerHTML = '';

    // Add options with checkboxes
    values.sort((a, b) => a.localeCompare(b)).forEach(value => {
        const li = document.createElement('li');
        const checkbox = document.createElement('input');
        const label = document.createElement('label');

        checkbox.type = 'checkbox';
        checkbox.value = value;
        checkbox.className = 'dropdown-checkbox';
        checkbox.onchange = filterData;

        label.textContent = value || '(空白)';
        label.style.marginLeft = '5px';

        li.appendChild(checkbox);
        li.appendChild(label);
        dropdown.appendChild(li);
    });
}

function filterDropdown(dropdownId) {
    const searchInput = document.getElementById(`${dropdownId}Search`).value.toLowerCase();
    const options = document.querySelectorAll(`#${dropdownId}Dropdown li`);

    options.forEach(option => {
        const text = option.textContent.toLowerCase();
        option.style.display = text.includes(searchInput) ? '' : 'none';
    });
}

function filterData() {
    const selectedCompanyIds = getSelectedValues('companyId');
    const selectedAgreementDates = getSelectedValues('agreementDate');
    const selectedBranches = getSelectedValues('branch');
    const selectedDepartments = getSelectedValues('department');
    const selectedCompanyNames = getSelectedValues('companyName');

    const filteredData = filterDataArray.filter(item => {
        return (
            (selectedCompanyIds.length === 0 || selectedCompanyIds.includes(item.company_id)) &&
            (selectedAgreementDates.length === 0 || selectedAgreementDates.includes(item.sign_date)) &&
            (selectedBranches.length === 0 || selectedBranches.includes(item.tantou_shiten)) &&
            (selectedDepartments.length === 0 || selectedDepartments.includes(item.tantou_bumon)) &&
            (selectedCompanyNames.length === 0 || selectedCompanyNames.includes(item.company_name))
        );
    });

    displayResults(filteredData);
}

function getSelectedValues(dropdownId) {
    const checkboxes = document.querySelectorAll(`#${dropdownId}Dropdown .dropdown-checkbox`);
    return Array.from(checkboxes)
        .filter(checkbox => checkbox.checked)
        .map(checkbox => checkbox.value);
}
 
