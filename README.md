const response = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getProjectList",
            data: { _token: CSRF_TOKEN, folderId: BOX_MAIN_FOLDER_ID }
        });

        // サーバーからのレスポンスを分割
        const projects = response.projects;
        const loginStatus = response.login_status;

        // =================================================================
        //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
        //
        //  **【重要】ログイン状態に応じた処理**
        //
        if (loginStatus === 'logged_out') {
            alert('BOXにログインされていないため、更新データは取得できません。前回同期したデータを表示します。');
        }
        //
        //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
        // =================================================================
        
        // プロジェクトリストが空でもエラーにせず、ドロップダウンを空にするだけ
        modelSelector.innerHTML = '';
        
        if (Array.isArray(projects) && projects.length > 0) {
            projects.forEach(project => {
                const option = document.createElement('option');
                option.value = project.id;
                option.textContent = project.name;
                modelSelector.appendChild(option);
            });
            
            // プロジェクトリストがあれば、最初のモデルの読み込みを開始
            const modelToLoad = projects.length > 1 ? projects[1] : projects[0];
            initiateLoadProcess(modelToLoad.id, modelToLoad.name);

        } else {
            // Boxにログインしておらず、プロジェクトリストも取得できない場合
            if (loaderTextElement) loaderTextElement.textContent = "プロジェクトリストを取得できません。Boxにログインして再度お試しください。";
            if (loaderContainer) loaderContainer.style.display = 'none'; // ローディングを隠す
        }
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "プロジェクトリストの取得中にエラーが発生しました。";
    }
}
