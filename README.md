![image](https://github.com/user-attachments/assets/6f3f82b0-35f7-4e93-9194-cf70a2aa589a)

 
 <details open>
                            <summary><strong>会社ID</strong></summary>
                            <select id="companyId" onchange="filterData()" class="form-control">
                                <option value="">すべて</option>
                            </select>
                                                    </details>
                                                    <details open>
                                                        <summary><strong>協定書締結日</strong></summary>
                                                        <select id="agreementDate" onchange="filterData()" class="form-control">
                                                            <option value="">すべて</option>
                                                        </select>
                                                    </details>
                                                    
                                                    <details open>
                                                        <summary><strong>担当支店</strong></summary>
                                                        <select id="branch" onchange="filterData()" class="form-control">
                                                            <option value="">すべて</option>
                                                        </select>
                                                    </details>
                                                    <details open>
                                                        <summary><strong>担当部門</strong></summary>
                                                        <select id="department" onchange="filterData()" class="form-control">
                                                            <option value="">すべて</option>
                                                        </select>
                                                    </details>
                                                     <details open>
                            <summary><strong>会社名</strong></summary>
                            <select id="companyName" onchange="filterData()" class="form-control">
                                <option value="">すべて</option>
                            </select>
                        </details>


function populateFilters() {
    const companyIdFilter = document.getElementById('companyId');
    const agreementDateFilter = document.getElementById('agreementDate');
    const branchFilter = document.getElementById('branch');
    const departmentFilter = document.getElementById('department');
    const companyNameFilter = document.getElementById('companyName');

    const companyId = [...new Set(filterDataArray.map(item => item.company_id))];
    const agreementDate = [...new Set(filterDataArray.map(item => item.sign_date))];
    const branches = [...new Set(filterDataArray.map(item => item.tantou_shiten))];
    const departments = [...new Set(filterDataArray.map(item => item.tantou_bumon))];
    const companyNames = [...new Set(filterDataArray.map(item => item.company_name))];


    companyId.sort((a, b) => a - b).forEach(id => {
        const option = document.createElement('option');
        option.value = id;
        option.textContent = id;
        companyIdFilter.appendChild(option);
    });

    agreementDate.sort((a, b) => a - b).forEach(date => {
        const option = document.createElement('option');
        option.value = date;
        option.textContent = date;
        agreementDateFilter.appendChild(option);
    });

    branches.sort((a, b) => a.localeCompare(b)).forEach(branch => {
        const option = document.createElement('option');
        option.value = branch;
        option.textContent = branch;
        branchFilter.appendChild(option);
    });

    departments.sort((a, b) => a.localeCompare(b)).forEach(department => {
        const option = document.createElement('option');
        option.value = department;
        option.textContent = department;
        departmentFilter.appendChild(option);
    });

    companyNames.sort((a, b) => a.localeCompare(b)).forEach(companyName => {
        const option = document.createElement('option');
        option.value = companyName;
        option.textContent = companyName;
        companyNameFilter.appendChild(option);
    });

}

export function filterData() {
    const companyIdFilter = document.getElementById('companyId').value;
    const agreementDateFilter = document.getElementById('agreementDate').value;
    const branchFilter = document.getElementById('branch').value;
    const departmentFilter = document.getElementById('department').value;
    const companyNameFilter = document.getElementById('companyName').value;

    const filteredData = filterDataArray.filter(item => {
        return (companyIdFilter === '' || item.company_id === Number(companyIdFilter)) &&
            (agreementDateFilter === '' || item.sign_date == agreementDateFilter) &&
            (branchFilter === '' || item.tantou_shiten === branchFilter) &&
            (departmentFilter === '' || item.tantou_bumon == departmentFilter) &&
            (companyNameFilter === '' || item.company_name == companyNameFilter);
    });

    updateFilters(filteredData);
    displayResults(filteredData);
}

function updateFilters(filteredData) {
    const companyIdFilter = document.getElementById('companyId');
    const agreementDateFilter = document.getElementById('agreementDate');
    const branchFilter = document.getElementById('branch');
    const departmentFilter = document.getElementById('department');
    const companyNameFilter = document.getElementById('companyName');

    const selectedCompanyId = companyIdFilter.value;
    const selectedAgreementDate = agreementDateFilter.value;
    const selectedBranch = branchFilter.value;
    const selectedDepartment = departmentFilter.value;
    const selectedcompanyName = companyNameFilter.value;

    companyIdFilter.innerHTML = '<option value="">すべて</option>';
    agreementDateFilter.innerHTML = '<option value="">すべて</option>';
    branchFilter.innerHTML = '<option value="">すべて</option>';
    departmentFilter.innerHTML = '<option value="">すべて</option>';
    companyNameFilter.innerHTML = '<option value="">すべて</option>';

    const companyIds = [...new Set(filteredData.map(item => item.company_id))];
    const agreementDates = [...new Set(filteredData.map(item => item.sign_date))];
    const branches = [...new Set(filteredData.map(item => item.tantou_shiten))];
    const departments = [...new Set(filteredData.map(item => item.tantou_bumon))];
    const companyNames = [...new Set(filteredData.map(item => item.company_name))];

    companyIds.forEach(companyId => {
        const option = document.createElement('option');
        option.value = companyId;
        option.textContent = companyId;
        companyIdFilter.appendChild(option);
    });

    agreementDates.forEach(agreementDate => {
        const option = document.createElement('option');
        option.value = agreementDate;
        option.textContent = agreementDate;
        agreementDateFilter.appendChild(option);
    });

    branches.forEach(branch => {
        const option = document.createElement('option');
        option.value = branch;
        option.textContent = branch;
        branchFilter.appendChild(option);
    });

    departments.forEach(department => {
        const option = document.createElement('option');
        option.value = department;
        option.textContent = department;
        departmentFilter.appendChild(option);
    });

    companyNames.forEach(companyName => {
        const option = document.createElement('option');
        option.value = companyName;
        option.textContent = companyName;
        companyNameFilter.appendChild(option);
    });

    if (selectedCompanyId) companyIdFilter.value = selectedCompanyId;
    if (selectedAgreementDate) agreementDateFilter.value = selectedAgreementDate;
    if (selectedBranch) branchFilter.value = selectedBranch;
    if (selectedDepartment) departmentFilter.value = selectedDepartment;
    if (selectedcompanyName) companyNameFilter.value = selectedcompanyName;
}

the above code is original code
I want to modify this code with thw image design
1. add search function
2. add checkbox beside the option value reference Image
