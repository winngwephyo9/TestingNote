
<details open>
    <summary><strong>会社ID</strong></summary>
    <div class="custom-select-container">
        <button
            id="companyIdButton"
            onclick="toggleDropdown('companyId')"
            class="form-control dropdown-toggle"
        >
            すべて
        </button>
        <div id="companyIdDropdownContainer" class="dropdown-container" style="display: none;">
            <input
                type="text"
                id="companyIdSearch"
                onkeyup="filterDropdown('companyId')"
                placeholder="検索"
                class="form-control search-box"
            />
            <ul id="companyIdDropdown" class="dropdown-list">
                <!-- Items will be dynamically populated -->
            </ul>
        </div>
    </div>
</details>

<details open>
    <summary><strong>協定書締結日</strong></summary>
    <div class="custom-select-container">
        <button
            id="agreementDateButton"
            onclick="toggleDropdown('agreementDate')"
            class="form-control dropdown-toggle"
        >
            すべて
        </button>
        <div id="agreementDateDropdownContainer" class="dropdown-container" style="display: none;">
            <input
                type="text"
                id="agreementDateSearch"
                onkeyup="filterDropdown('agreementDate')"
                placeholder="検索"
                class="form-control search-box"
            />
            <ul id="agreementDateDropdown" class="dropdown-list">
                <!-- Items will be dynamically populated -->
            </ul>
        </div>
    </div>
</details>

.custom-select-container {
    position: relative;
}

.dropdown-toggle {
    cursor: pointer;
    text-align: left;
    width: 100%;
}

.dropdown-container {
    position: absolute;
    background-color: white;
    border: 1px solid #ccc;
    max-height: 200px;
    overflow-y: auto;
    z-index: 1000;
    width: 100%;
}

.dropdown-list {
    list-style: none;
    margin: 0;
    padding: 0;
}

.dropdown-list li {
    padding: 5px;
    cursor: pointer;
}

.dropdown-list li:hover {
    background-color: #f0f0f0;
}

.search-box {
    margin: 5px 0;
    padding: 5px;
    width: 100%;
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
    dropdown.innerHTML = `
        <li>
            <input
                type="radio"
                id="${dropdownId}_all"
                name="${dropdownId}"
                value=""
                class="dropdown-radio"
                onchange="selectOption('${dropdownId}', 'すべて')"
                checked
            />
            <label for="${dropdownId}_all">すべて</label>
        </li>
    `;

    // Add other options with radio buttons
    values.sort((a, b) => a.localeCompare(b)).forEach((value, index) => {
        const id = `${dropdownId}_${index}`;
        const li = document.createElement('li');

        const radio = document.createElement('input');
        radio.type = 'radio';
        radio.id = id;
        radio.name = dropdownId;
        radio.value = value;
        radio.className = 'dropdown-radio';
        radio.onchange = () => selectOption(dropdownId, value);

        const label = document.createElement('label');
        label.textContent = value || '(空白)';
        label.htmlFor = id;

        li.appendChild(radio);
        li.appendChild(label);
        dropdown.appendChild(li);
    });
}

function toggleDropdown(dropdownId) {
    const dropdownContainer = document.getElementById(`${dropdownId}DropdownContainer`);
    dropdownContainer.style.display =
        dropdownContainer.style.display === 'none' ? 'block' : 'none';
}

function selectOption(dropdownId, value) {
    const button = document.getElementById(`${dropdownId}Button`);
    button.textContent = value || 'すべて';

    // Close dropdown after selection
    const dropdownContainer = document.getElementById(`${dropdownId}DropdownContainer`);
    dropdownContainer.style.display = 'none';

    // Trigger filter update
    filterData();
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
    const selectedCompanyId = getSelectedValue('companyId');
    const selectedAgreementDate = getSelectedValue('agreementDate');
    const selectedBranch = getSelectedValue('branch');
    const selectedDepartment = getSelectedValue('department');
    const selectedCompanyName = getSelectedValue('companyName');

    const filteredData = filterDataArray.filter(item => {
        return (
            (selectedCompanyId === '' || item.company_id === selectedCompanyId) &&
            (selectedAgreementDate === '' || item.sign_date === selectedAgreementDate) &&
            (selectedBranch === '' || item.tantou_shiten === selectedBranch) &&
            (selectedDepartment === '' || item.tantou_bumon === selectedDepartment) &&
            (selectedCompanyName === '' || item.company_name === selectedCompanyName)
        );
    });

    displayResults(filteredData);
}

function getSelectedValue(dropdownId) {
    const selectedRadio = document.querySelector(`#${dropdownId}Dropdown .dropdown-radio:checked`);
    return selectedRadio ? selectedRadio.value : '';
}

<!-- Repeat similar structure for branch, department, and companyName -->
