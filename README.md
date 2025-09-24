<header>
    <div class="header-left">
        <!-- ... Select Modelなどの要素 ... -->
    </div>
    <div class="header-right">
        <!-- 
            【新規】
            Boxのログイン状態に応じてメッセージを表示するためのコンテナを追加。
            最初は非表示にしておく。
        -->
        <div id="box-status-message" class="box-status-info" style="display: none;">
            <span class="info-icon">ℹ️</span>
            <span id="box-status-text"></span>
        </div>

        <!-- ... BOX LOGOUTボタンなどの要素 ... -->
        <div id="box_login_timer_box" class="boxtimer">
            BOXログイン有効時間<div id="box_login_timer" class="boxtimer-time"></div>
        </div>
    </div>
</header>


/* Boxステータスメッセージ用のスタイル */
.box-status-info {
    display: flex; /* アイコンとテキストを横並びにする */
    align-items: center; /* 上下中央揃え */
    padding: 8px 12px;
    margin-right: 20px;
    border-radius: 5px;
    font-size: 14px;
    background-color: #e6f7ff; /* 薄い青色 */
    border: 1px solid #91d5ff; /* 少し濃い青色 */
    color: #00528e; /* 濃い青色のテキスト */
}

.box-status-info .info-icon {
    margin-right: 8px;
    font-size: 16px;
}```

---

### ステップ3：JavaScriptの修正 (`objViewerStandard.js`)

`initiateProjectPopulation`関数を修正し、サーバーからの`login_status`に応じて、アラートの代わりにこの新しいメッセージ要素の表示を制御するようにします。

**このファイルのみ修正が必要です。**

```javascript
// ... import文 ...

// --- Get UI Elements ---
// ...
const boxStatusMessage = document.getElementById('box-status-message'); // 【新規】メッセージコンテナを取得
const boxStatusText = document.getElementById('box-status-text');       // 【新規】メッセージテキスト部分を取得

// ... グローバル変数、シーン設定などは変更なし ...

/**
 * 【最終修正版】ログイン状態に応じてナビゲーションバーにメッセージを表示
 */
async function initiateProjectPopulation() {
    // ページロード時に、まずメッセージを隠しておく
    if (boxStatusMessage) boxStatusMessage.style.display = 'none';

    try {
        const response = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getProjectList",
            data: { _token: CSRF_TOKEN, folderId: BOX_MAIN_FOLDER_ID }
        });

        const projects = response.projects;
        const loginStatus = response.login_status;

        // =================================================================
        //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
        //
        //  **【重要】ログイン状態に応じてメッセージの表示を制御**
        //
        if (loginStatus === 'logged_out') {
            // 未ログインの場合、メッセージを表示
            if (boxStatusMessage && boxStatusText) {
                boxStatusText.textContent = 'Boxに未ログインです。キャッシュされたデータを表示中。最新データはBoxにログインしてください。';
                boxStatusMessage.style.display = 'flex'; // flexで表示
            }
        }
        //
        //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
        // =================================================================
        
        modelSelector.innerHTML = '';
        
        if (Array.isArray(projects) && projects.length > 0) {
            projects.forEach(project => {
                const option = document.createElement('option');
                option.value = project.id;
                option.textContent = project.name;
                modelSelector.appendChild(option);
            });
            
            const modelToLoad = projects[0];
            startSync(modelToLoad.id, modelToLoad.name);

        } else {
            if (loaderTextElement) loaderTextElement.textContent = "表示できるプロジェクトがありません。";
            hideLoading();
            // プロジェクトがない場合、未ログインメッセージも隠す
            if (boxStatusMessage) boxStatusMessage.style.display = 'none';
        }
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "プロジェクトリストの取得中にエラーが発生しました。";
        hideLoading();
    }
}


/**
 * 【修正版】startSyncからアラートを削除
 */
function startSync(projectFolderId, projectName) {
    if (currentLoadedProjectId === projectFolderId && !loaderContainer.classList.contains('show')) return;
    currentLoadedProjectId = projectFolderId;

    showLoading();
    if (loaderTextElement) loaderTextElement.textContent = `サーバーと同期を開始しています...`;
    
    $.ajax({
        type: "post",
        url: url_prefix + "/box/startSync",
        data: { _token: CSRF_TOKEN, folderId: projectFolderId },
        success: function (response) {
            if (response.status === 'no_token' || response.token === 'no_token') {
                // **アラートはinitiateProjectPopulationで表示済みなので、ここでは何もしない**
                // そのままキャッシュ表示を試みる
                fetchAndRenderModel(projectFolderId, projectName); 
            } else if (response.status === 'job_dispatched') {
                if (loaderTextElement) loaderTextElement.textContent = `サーバーで同期中です...`;
                checkSyncStatus(projectFolderId, projectName);
            }
        },
        error: function(err) {
            // ...
        }
    });
}

// ... これ以降のJavaScriptコード (checkSyncStatus, fetchAndRenderModelなど) は変更ありません ...
