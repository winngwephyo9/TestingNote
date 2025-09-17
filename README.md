    //  **【重要】不要な再読み込みを防ぐガード節**
    //
    // もし今からロードしようとしているプロジェクトが、既にロード済みのプロジェクトで、
    // かつ、バックグラウンドでの同期処理が走っていないなら、処理を中断する。
    //
    if (currentLoadedProjectId === projectFolderId && !syncPollingInterval) {
        console.log(`Project ${projectName} is already loaded. Skipping reload.`);
        return; // ここで処理を中断
    }
