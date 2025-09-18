/**
 * 【修正版】モデルの描画とポーリング要否の判断
 */
async function loadModel(projectFolderId, projectName) {
    resetScene();
    if (loaderContainer) loaderContainer.style.display = 'flex';

    try {
        if (loaderTextElement) loaderTextElement.textContent = `Fetching model data for ${projectName}...`;
        const response = await $.ajax({/* ... */});
        const filePairs = response.objMtlPairs; 
        const syncStatus = response.sync_status;

        if (!Array.isArray(filePairs) || filePairs.length === 0) {
            if (syncStatus === 'processing') {
                if (loaderTextElement) loaderTextElement.textContent = `サーバーで初回同期中です... この処理には数十分かかる場合があります。`;
                currentLoadedProjectId = projectFolderId;
                return true; 
            }
            throw new Error(response.error || `No model data found.`);
        }
        
        // ... (モデルの解析・表示ロジックは変更なし) ...
        
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        updateInfoPanel();
        
        currentLoadedProjectId = projectFolderId;

        // 【重要】ポーリングが不要なら、ここでローディングを完了させる
        if (syncStatus !== 'processing') {
            if (loaderContainer) loaderContainer.style.display = 'none';
        }
        return syncStatus === 'processing';

    } catch (error) {
        if (loaderTextElement) loaderTextElement.textContent = `Error: ${error.message}`;
        currentLoadedProjectId = null;
        if (loaderContainer) loaderContainer.style.display = 'none'; // エラー時も隠す
        return false;
    }
}
