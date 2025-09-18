/**
 * 【修正版】同期状態をサーバーに定期的に問い合わせる
 */
function startSyncStatusPolling(projectFolderId, projectName) {
    if (syncPollingInterval) clearInterval(syncPollingInterval);
    if (loaderContainer) loaderContainer.style.display = 'flex';
    
    syncPollingInterval = setInterval(async () => {
        // 【重要】ポーリングの問い合わせ先は、現在選択されているモデルID（modelSelector.value）にする
        const currentlySelectedId = modelSelector.value;
        if (!currentlySelectedId) return; // ドロップダウンが空なら何もしない

        try {
            const response = await $.ajax({
                type: "post",
                url: url_prefix + "/box/getSyncStatus",
                data: { _token: CSRF_TOKEN, folderId: currentlySelectedId }
            });

            if (response.sync_status === 'completed' || response.sync_status === 'failed') {
                clearInterval(syncPollingInterval);
                syncPollingInterval = null;
                
                if (loaderTextElement) loaderTextElement.textContent = `同期が完了しました。最新のモデルを再読み込みします...`;
                
                // 完了したのは現在選択中のモデルなので、それを再ロード
                const selectedOption = modelSelector.options[modelSelector.selectedIndex];
                initiateLoadProcess(selectedOption.value, selectedOption.text);
            }
        } catch (error) {
            console.error("Failed to get sync status:", error);
            clearInterval(syncPollingInterval);
            syncPollingInterval = null;
            if (loaderContainer) loaderContainer.style.display = 'none';
        }
    }, 10000);
}
