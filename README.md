 // ステップ1：UIの構築
    projects.forEach(project => {
        const option = document.createElement('option');
        option.value = project.id;
        option.textContent = project.name;
        modelSelector.appendChild(option);
    });
    
    // ステップ2：ロードするモデルを1つだけ決定
    const modelToLoad = projects[0]; // 常にリストの最初のモデルを選ぶ

    // ステップ3：ロード処理を一度だけ呼び出す
    initiateLoadProcess(modelToLoad.id, modelToLoad.name);
