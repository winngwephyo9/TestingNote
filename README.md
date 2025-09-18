// ... import文、グローバル変数、シーン設定などは変更なし ...

// =================================================================
//  【リファクタリング版】参考コードに合わせたシンプルなポーリングロジック
// =================================================================

$(document).ready(function () {
    // ...
    initiateProjectPopulation();
    modelSelector.addEventListener('change', modelSelectorChanged);
    // ...
});

async function initiateProjectPopulation() {
    try {
        const response = await $.ajax({ url: url_prefix + "/box/getProjectList", /* ... */ });
        const projects = response.projects;
        modelSelector.innerHTML = '';
        if (Array.isArray(projects) && projects.length > 0) {
            projects.forEach(project => {
                const option = document.createElement('option');
                option.value = project.id; option.textContent = project.name;
                modelSelector.appendChild(option);
            });
            const modelToLoad = projects[0];
            startSync(modelToLoad.id, modelToLoad.name);
        }
    } catch (error) { /* ... */ }
}

function modelSelectorChanged() {
    const selectedId = modelSelector.value;
    const selectedName = modelSelector.options[modelSelector.selectedIndex].text;
    startSync(selectedId, selectedName);
}

function startSync(projectFolderId, projectName) {
    showLoading();
    if (loaderTextElement) loaderTextElement.textContent = `サーバーと同期を開始しています...`;

    $.ajax({
        type: "post",
        url: url_prefix + "/box/startSync",
        data: { _token: CSRF_TOKEN, folderId: projectFolderId },
        success: function (response) {
            if (response.status === 'no_token' || response.token === 'no_token') {
                alert("BOXにログインされていないため更新できません。キャッシュされたデータを表示します。");
                fetchAndRenderModel(projectFolderId, projectName); 
            } else if (response.status === 'job_dispatched') {
                if (loaderTextElement) loaderTextElement.textContent = `サーバーで同期中です... この処理には数分かかる場合があります。`;
                checkSyncStatus(projectFolderId, projectName);
            }
        },
        error: function(err) {
            alert("同期の開始に失敗しました。");
            hideLoading();
        }
    });
}

function checkSyncStatus(projectFolderId, projectName) {
    setTimeout(async () => {
        try {
            const response = await $.ajax({
                type: "post",
                url: url_prefix + "/box/checkSyncJobStatus",
                data: { _token: CSRF_TOKEN, folderId: projectFolderId }
            });

            if (response.status === 'completed' || response.status === 'no_files_to_update') {
                const message = response.status === 'completed' ? '同期が完了しました。' : 'データは最新です。';
                if (loaderTextElement) loaderTextElement.textContent = `${message} モデルを読み込みます...`;
                fetchAndRenderModel(projectFolderId, projectName);
            } else if (response.status === 'failed') {
                alert(`同期に失敗しました: ${response.message}`);
                // 失敗した場合でも、古いキャッシュの表示を試みる
                fetchAndRenderModel(projectFolderId, projectName);
            } else { // processing
                checkSyncStatus(projectFolderId, projectName);
            }
        } catch (error) {
            alert("同期状態の確認に失敗しました。");
            hideLoading();
        }
    }, 5000); // 5秒後
}

async function fetchAndRenderModel(projectFolderId, projectName) {
    resetScene();
    if (loaderTextElement) loaderTextElement.textContent = `モデルデータを取得しています...`;
    try {
        const filePairs = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getModelData",
            data: { _token: CSRF_TOKEN, folderId: projectFolderId }
        });

        if (!Array.isArray(filePairs) || filePairs.length === 0) {
            throw new Error(`表示できるモデルデータがありません。`);
        }

        if (loaderTextElement) loaderTextElement.textContent = `モデルを構築しています...`;
        // ... ここに、以前のloadModelにあった3Dモデルの解析・結合・表示ロジックをすべて含める ...
        
        hideLoading();
    } catch (error) {
        if (loaderTextElement) loaderTextElement.textContent = `Error: ${error.message}`;
        hideLoading();
    }
}

function showLoading() { /* ... */ }
function hideLoading() { /* ... */ }
// ... resetSceneなどのヘルパー関数 ...
