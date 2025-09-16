// ... グローバル変数と初期設定は変更なし ...

async function loadModel(projectFolderId, projectName) {
    if (syncPollingInterval) {
        clearInterval(syncPollingInterval);
        syncPollingInterval = null;
    }
    
    resetScene();
    if (loaderContainer) loaderContainer.style.display = 'flex';

    try {
        if (loaderTextElement) loaderTextElement.textContent = `Fetching model data for ${projectName}...`;
        
        const response = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getModelData",
            data: { _token: CSRF_TOKEN, folderId: projectFolderId }
        });

        // =================================================================
        //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
        //
        //  **【重要】レスポンスの構造に合わせてデータを取得**
        //
        const filePairs = response.objMtlPairs; 
        const syncStatus = response.sync_status;
        //
        //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
        // =================================================================

        if (syncStatus === 'processing') {
            if (loaderTextElement) loaderTextElement.textContent = `サーバーで大規模なモデル同期中です... この処理には数十分かかる場合があります。`;
            // キャッシュデータがあれば、それをまず表示する
            if (filePairs && filePairs.length > 0) {
                if (loaderTextElement) loaderTextElement.textContent += ` (現在、前回同期したデータを表示しています)`;
                // この後の解析処理に進む
            } else {
                // キャッシュもない場合はポーリングを開始して待つ
                startSyncStatusPolling(projectFolderId, projectName);
                return; 
            }
        }
        if (syncStatus === 'failed') {
            // 失敗した場合でも、キャッシュデータがあれば表示を試みる
            if (loaderTextElement) loaderTextElement.textContent = `前回の同期に失敗しました。管理者にご確認ください。`;
        }
        
        // --- ここからモデルのロード処理 ---
        if (!Array.isArray(filePairs) || filePairs.length === 0) {
            // 同期中でなく、キャッシュもない場合はエラー
            if (syncStatus !== 'processing') {
                throw new Error(response.error || `No model data found for project "${projectName}".`);
            }
            return;
        }
        
        if (loaderTextElement) loaderTextElement.textContent = `Processing geometry and materials...`;

        // ... この後の解析、結合、スケーリング、カテゴリ取得、UI構築のロジックは全て変更ありません ...
        
        const allLoadedObjects = [];
        const firstObjContent = filePairs.find(p => p.obj)?.obj.content;
        if (firstObjContent) {
            const headerData = await parseObjHeader(firstObjContent);
            if (headerData) {
                parsedWSCenID = headerData.wscenId;
                parsedPJNo = headerData.pjNo;
            }
        }
        
        // ... (forループでの解析処理) ...
        
        // ... (ジオメトリ検証と結合) ...
        
        // ... (スケーリングと回転) ...

        // ... (カテゴリデータ取得) ...
        
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        updateInfoPanel();
        
        // もし裏で同期が進んでいるなら、ポーリングを開始する
        if(syncStatus === 'processing'){
            startSyncStatusPolling(projectFolderId, projectName);
        }

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error.responseText || error);
        let errorMessage = `Error: ${error.message || 'An unknown error occurred'}.`;
        if (error.responseJSON && error.responseJSON.error) {
            errorMessage = `Error: ${error.responseJSON.error}`;
        }
        if (loaderTextElement) loaderTextElement.textContent = errorMessage;
    } finally {
        // ポーリング中でなければローダーを隠す
        if (!syncPollingInterval) {
            if (loaderContainer) loaderContainer.style.display = 'none';
        }
    }
}
