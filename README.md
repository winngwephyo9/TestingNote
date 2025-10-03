// objViewerStandard.js

/**
 * 【最終修正版】CSVとOBJの座標を加算してモデルを構築する
 */
async function fetchAndRenderModel(projectFolderId, projectName) {
    resetScene();
    showLoading();
    if (loaderTextElement) loaderTextElement.textContent = `モデルデータを取得しています...`;

    try {
        const response = await $.ajax({ /* ... getModelData API呼び出し ... */ });
        if (!response.csv_data || !response.obj_mtl_pairs) {
            throw new Error(`サーバーからのデータ形式が正しくありません。`);
        }
        
        const csvData = response.csv_data;
        const filePairs = response.obj_mtl_pairs;

        if (!csvData.content) throw new Error(`全体配置座標を記載したCSVファイルが見つかりません。`);
        if (!Array.isArray(filePairs) || filePairs.length === 0) throw new Error(`表示できるOBJファイルがありません。`);

        if (loaderTextElement) loaderTextElement.textContent = `座標データを解析しています...`;

        // =================================================================
        //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
        //
        //  **【①, ②】CSVを解析し、複合キーで座標マップを作成**
        //
        const coordinatesMap = new Map();
        const csvLines = csvData.content.split(/\r?\n/);
        
        // ヘッダー行を特定（例: "タイプ名"が含まれる行）
        const headerIndex = csvLines.findIndex(line => line.includes('タイプ名'));
        const startRow = headerIndex !== -1 ? headerIndex + 1 : 0;

        for (let i = startRow; i < csvLines.length; i++) {
            const line = csvLines[i].trim();
            if (line === '') continue;
            
            const columns = line.split(/[ ,]+/); // スペースまたはカンマで分割
            
            const typeId = columns[1];
            const insGeoObjX = columns[2];
            const x = parseFloat(columns[3]);
            const y = parseFloat(columns[4]);
            const z = parseFloat(columns[5]);

            if (typeId && insGeoObjX && !isNaN(x) && !isNaN(y) && !isNaN(z)) {
                // 複合キーを作成: "タイプID-ジオメトリID" (例: "9396218-107")
                const compositeKey = `${typeId}-${insGeoObjX}`;
                coordinatesMap.set(compositeKey, new THREE.Vector3(x, y, z));
            }
        }
        console.log("Coordinates Map created with composite keys:", coordinatesMap);


        if (loaderTextElement) loaderTextElement.textContent = `モデルを構築しています...`;
        
        loadedObjectModelRoot = new THREE.Group();
        const objLoader = new OBJLoader();
        const mtlLoader = new MTLLoader();

        // **【③, ④】各OBJファイルを処理し、座標を加算してモデリング**
        for (const pair of filePairs) {
            try {
                if (!pair.obj.content || !pair.obj.content.trim()) continue;

                // OBJファイルからタイプIDとジオメトリIDを解析
                const objName = pair.obj.name;
                const typeIdMatch = objName.match(/_(\d+)\.obj$/);
                const typeId = typeIdMatch ? typeIdMatch[1] : null;

                const geoIdMatch = pair.obj.content.match(/# GeometryObject\.Id:(\d+)/);
                const insGeoObjX = geoIdMatch ? geoIdMatch[1] : null;
                
                if (!typeId || !insGeoObjX) {
                    console.warn(`Could not parse IDs from OBJ file: ${objName}. Skipping.`);
                    continue;
                }
                
                const compositeKey = `${typeId}-${insGeoObjX}`;
                const basePosition = coordinatesMap.get(compositeKey);

                if (!basePosition) {
                    console.warn(`Coordinates not found for key: ${compositeKey}. Skipping object.`);
                    continue;
                }

                // 頂点座標を書き換えた新しいOBJコンテンツを生成
                const objLines = pair.obj.content.split(/\r?\n/);
                const newObjContentLines = [];
                for (const line of objLines) {
                    if (line.startsWith('v ')) { // 頂点座標の行かチェック
                        const parts = line.split(/[ ]+/);
                        const relX = parseFloat(parts[1]);
                        const relY = parseFloat(parts[2]);
                        const relZ = parseFloat(parts[3]);
                        
                        // 座標を加算
                        const absX = basePosition.x + relX;
                        const absY = basePosition.y + relY;
                        const absZ = basePosition.z + relZ;
                        
                        // 新しい行を生成
                        newObjContentLines.push(`v ${absX} ${absY} ${absZ}`);
                    } else {
                        // 頂点以外の行はそのままコピー
                        newObjContentLines.push(line);
                    }
                }
                const newObjContent = newObjContentLines.join('\n');

                // MTLを解析
                let materialsCreator = null;
                if (pair.mtl && pair.mtl.content) {
                    materialsCreator = mtlLoader.parse(pair.mtl.content, '');
                }
                if (materialsCreator) objLoader.setMaterials(materialsCreator);
                
                // 座標が修正された新しいコンテンツでオブジェクトをパース
                const parsedGroup = objLoader.parse(newObjContent);
                
                // 正常にパースできたら、子オブジェクトをメインのルートに追加
                if (parsedGroup) {
                    while(parsedGroup.children.length > 0) {
                        loadedObjectModelRoot.add(parsedGroup.children[0]);
                    }
                }

            } catch (e) { console.error("Error processing OBJ/MTL pair:", { pair, error: e }); }
        }
        
        if (loadedObjectModelRoot.children.length === 0) {
            throw new Error("有効なオブジェクトを構築できませんでした。");
        }
        
        // 【重要】全体の中心移動やスケーリングは、必要に応じて調整
        // 基本的には、CSVの座標が絶対座標なので、中心移動は不要
        // loadedObjectModelRoot.position.sub(center); // ← 不要
        
        // 単位がmmの場合、mに変換するためのスケーリングなど
        const modelScale = 0.001;
        loadedObjectModelRoot.scale.set(modelScale, modelScale, modelScale);

        // Z-upからY-upへの変換など、必要であれば回転
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;
        
        //
        //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
        // =================================================================

        // ... (カテゴリデータ取得、モデルツリー構築、シーンへの追加、カメラのフレーミングは同じ) ...
        
        await buildAndPopulateCategorizedTree(); // モデルツリーのロジックは、オブジェクト名に依存
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        updateInfoPanel();
        
    } catch (error) {
        // ...
    } finally {
        hideLoading();
    }
}

// ... これ以降のヘルパー関数は変更ありません ...
