/**
 * 【最終修正版】
 * サーバーから取得したデータ（CSVとOBJペア）を元に、各OBJの頂点座標を
 * CSVの絶対座標で動的に計算し直し、3Dシーンを構築・描画する。
 * 
 * @param {string} projectFolderId - 現在のプロジェクトフォルダID
 * @param {string} projectName - 現在のプロジェクト名
 */
async function fetchAndRenderModel(projectFolderId, projectName) {
    resetScene();
    showLoading();
    if (loaderTextElement) loaderTextElement.textContent = `モデルデータを取得しています...`;

    try {
        // 1. サーバーからCSVデータとOBJ/MTLペアのリストを取得
        const response = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getModelData",
            data: { _token: CSRF_TOKEN, folderId: projectFolderId }
        });

        // 必要なデータ構造が存在するかチェック
        if (!response.csv_data || !response.obj_mtl_pairs) {
            throw new Error(`サーバーから返されたデータの形式が正しくありません。`);
        }
        
        const csvData = response.csv_data;
        const filePairs = response.obj_mtl_pairs;

        if (!csvData || !csvData.content) {
            throw new Error(`全体配置座標を記載したCSVファイルが見つかりません。`);
        }
        if (!Array.isArray(filePairs) || filePairs.length === 0) {
            throw new Error(`表示できるOBJファイルがありません。`);
        }

        if (loaderTextElement) loaderTextElement.textContent = `座標データを解析しています...`;

        // 2. CSVを解析し、複合キー("タイプID-ジオメトリID")をキーとする座標マップを作成
        const coordinatesMap = new Map();
        const csvLines = csvData.content.split(/\r?\n/);
        
        // ヘッダー行を特定し、それ以降の行から処理を開始
        const headerIndex = csvLines.findIndex(line => line.includes('タイプ名') || line.includes('タイプ_ID'));
        const startRow = headerIndex !== -1 ? headerIndex + 1 : 0;

        for (let i = startRow; i < csvLines.length; i++) {
            const line = csvLines[i].trim();
            if (line === '') continue;
            
            const columns = line.split(/[ ,]+/); // スペースまたはカンマ区切りに対応
            
            const typeId = columns[1];      // タイプ_ID
            const insGeoObjX = columns[2];  // InsGeoObX
            const x = parseFloat(columns[3]);
            const y = parseFloat(columns[4]);
            const z = parseFloat(columns[5]);

            if (typeId && insGeoObjX && !isNaN(x) && !isNaN(y) && !isNaN(z)) {
                const compositeKey = `${typeId}-${insGeoObjX}`;
                coordinatesMap.set(compositeKey, new THREE.Vector3(x, y, z));
            }
        }

        if (coordinatesMap.size === 0) {
            console.warn("No valid coordinates found in the CSV file.");
        }

        if (loaderTextElement) loaderTextElement.textContent = `モデルを構築しています... (${filePairs.length}個のオブジェクト)`;
        
        loadedObjectModelRoot = new THREE.Group();
        const objLoader = new OBJLoader();
        const mtlLoader = new MTLLoader();

        // 3. 各OBJファイルを処理し、座標を加算してモデリング
        for (const pair of filePairs) {
            try {
                if (!pair.obj || !pair.obj.content || !pair.obj.content.trim()) continue;

                // OBJファイル名からタイプIDを抽出 (例: ..._9396218.obj)
                const typeIdMatch = pair.obj.name.match(/_(\d+)\.obj$/);
                const typeId = typeIdMatch ? typeIdMatch[1] : null;

                // OBJファイルの中身からジオメトリIDを抽出 (例: # GeometryObject.Id:107)
                const geoIdMatch = pair.obj.content.match(/#\s*GeometryObject\.Id\s*:\s*(\d+)/);
                const insGeoObjX = geoIdMatch ? geoIdMatch[1] : null;
                
                if (!typeId || !insGeoObjX) {
                    console.warn(`Could not parse TypeID or GeometryID from OBJ file: ${pair.obj.name}. Skipping.`);
                    continue;
                }
                
                const compositeKey = `${typeId}-${insGeoObjX}`;
                const basePosition = coordinatesMap.get(compositeKey);

                if (!basePosition) {
                    console.warn(`Coordinates not found in CSV for key: ${compositeKey} (from file ${pair.obj.name}). Skipping object.`);
                    continue;
                }

                // 頂点座標を書き換えた新しいOBJコンテンツ（文字列）を生成
                const objLines = pair.obj.content.split(/\r?\n/);
                const newObjContentLines = [];
                for (const line of objLines) {
                    if (line.trim().startsWith('v ')) { // 頂点座標の行かチェック
                        const parts = line.trim().split(/[ ]+/); // トリムしてから分割
                        const relX = parseFloat(parts[1]);
                        const relY = parseFloat(parts[2]);
                        const relZ = parseFloat(parts[3]);
                        
                        // 座標を加算
                        const absX = basePosition.x + relX;
                        const absY = basePosition.y + relY;
                        const absZ = basePosition.z + relZ;
                        
                        newObjContentLines.push(`v ${absX} ${absY} ${absZ}`);
                    } else {
                        // 頂点以外の行（mtllib, g, usemtl, f など）はそのままコピー
                        newObjContentLines.push(line);
                    }
                }
                const newObjContent = newObjContentLines.join('\n');

                // MTL（マテリアル）を解析
                let materialsCreator = null;
                if (pair.mtl && pair.mtl.content) {
                    materialsCreator = mtlLoader.parse(pair.mtl.content, '');
                }
                if (materialsCreator) {
                    objLoader.setMaterials(materialsCreator);
                }
                
                // 座標が修正された新しいコンテンツで3Dオブジェクトをパース
                const parsedGroup = objLoader.parse(newObjContent);
                
                if (parsedGroup) {
                    // パースして得られたオブジェクト（通常は1つ）をメインのルートに追加
                    // この時、オブジェクト名はOBJファイル内の 'g' タグから引き継がれる
                    while(parsedGroup.children.length > 0) {
                        loadedObjectModelRoot.add(parsedGroup.children[0]);
                    }
                }

            } catch (e) { 
                console.error("Error processing an individual OBJ/MTL pair:", { name: pair.obj.name, error: e }); 
            }
        }
        
        if (loadedObjectModelRoot.children.length === 0) {
            throw new Error("有効な3Dオブジェクトを一つも構築できませんでした。CSVとOBJのIDが一致しているか確認してください。");
        }
        
        // 単位がmmの場合、mに変換するためのスケーリングなど、必要に応じて調整
        const modelScale = 0.001;
        loadedObjectModelRoot.scale.set(modelScale, modelScale, modelScale);

        // Z-upからY-upへの変換 (Revitからの出力などで一般的)
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;
        
        // --- 後処理 ---
        const allIds = [...new Set(loadedObjectModelRoot.children.map(c => {
            const s = Math.max(c.name.lastIndexOf('_'), c.name.lastIndexOf('＿'));
            return s > 0 ? c.name.substring(s + 1) : null;
        }).filter(Boolean))];
        
        await fetchAllCategoryData(parsedWSCenID, allIds);
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        updateInfoPanel();
        
    } catch (error) {
        console.error("Failed to render model:", error);
        if (loaderTextElement) loaderTextElement.textContent = `モデルの表示に失敗しました: ${error.message}`;
    } finally {
        hideLoading();
    }
}
