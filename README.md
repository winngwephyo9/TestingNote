<img width="1248" height="705" alt="image" src="https://github.com/user-attachments/assets/cc7cd3d9-7c29-466d-98d3-986eec4dd72d" />


はい、承知いたしました。課長様からのご指示、画像を拝見し、新しいOBJファイルの読み込み方法を完全に理解しました。

これは、3Dデータの構成における非常に重要で、より高度なアーキテクチャへの変更です。ご安心ください、現在のコードをベースに、この新しい方式に対応するための修正方法を詳しく解説します。

1. 新しいアーキテクチャのコンセプト（課長様のご指示の解説）

まず、何が変わるのかを正確に理解しましょう。

以前の方法（バラバラのOBJ）:

各OBJファイルは、それ自体がワールド空間内の正しい絶対座標に配置されていると想定されていました。

そのため、プログラムはすべてのOBJを読み込んだ後、モデル全体の中心を計算し、それをシーンの中心に移動させていました (loadedObjectModelRoot.position.sub(center))。

新しい方法（CSV + 個別OBJ）:

個別のOBJファイル: 各OBJファイル（部材）は、それ自体のローカルな原点（0, 0, 0）周りにモデリングされています。これらのファイルは、自分が最終的にどこに配置されるかを知りません。

全体配置座標のCSV: これが「設計図」や「組立指示書」の役割を果たします。このCSVファイルに、「どの部材IDのOBJを、どの絶対座標XYZに配置するか」という情報がすべて記載されています。

つまり、これからの処理は「たくさんの部品（個別OBJ）と、一つの組立指示書（CSV）を受け取って、3D空間でプラモデルのように組み立てる」という流れに変わります。

この変更によるメリット:

データ管理の効率化: 部品が一つ変更されても、その小さなOBJファイルを差し替えるだけで済みます。全体の巨大なモデルを再生成する必要がありません。

再利用性の向上: 同じ形状の部品（例えば同じボルト）を複数の場所に配置したい場合、OBJファイルは1つだけで、CSVに行を複数追加するだけで対応できます。

柔軟性: 3Dモデルのレイアウト変更が、CSVというテキストデータを編集するだけで可能になり、非常に柔軟になります。

2. 実装のための修正ステップ

この新しいアーキテクチャを実装するために、Box上のファイル構成、バックエンド（コントローラー/モデル）、そして**フロントエンド（JavaScript）**の3つを修正する必要があります。

ステップ1：Box上のファイル構成の変更

各プロジェクトフォルダの中に、これまでのOBJ/MTLファイルに加えて、全体配置座標を記載したCSVファイルを1つだけ配置する必要があります。
例: layout.csv, coordinates.csv など、決まった名前のファイル。

ステップ2：バックエンドの修正 (DLDHWDataImportModel.php と Controller)

サーバーは、フロントエンドに対して、OBJ/MTLペアのリストだけでなく、この重要なCSVファイルの情報も渡す必要があります。

DLDHWDataImportModel.php の syncAndGetModelData の修正
このメソッドは、contentをDBに保存する方式のままで、CSVファイルも他のOBJ/MTLと同様にmodel_file_cacheテーブルに保存するようにします。特別な変更はほとんど不要です。

DLDHWDataImportModel.php の getCachedModelData の修正
DBからデータを取得する際に、CSVファイルとOBJ/MTLペアを区別してフロントエンドに返すようにします。

code
PHP
download
content_copy
expand_less
// DLDHWDataImportModel.php
public function getCachedModelData($projectFolderId)
{
    $cachedFiles = ModelFileCache::where('project_box_id', $projectFolderId)->get();
    if ($cachedFiles->isEmpty()) {
        return ['error' => 'No cached model found.'];
    }

    // CSVファイルを探す（ファイル名で判断）
    $csvFile = $cachedFiles->first(function ($file) {
        return strtolower($file->file_type) === 'csv';
    });

    // OBJ/MTLファイルをペアにする
    $objMtlPairs = [];
    $groupedFiles = $cachedFiles->where('file_type', '!=', 'csv')->groupBy('base_name');
    foreach ($groupedFiles as $baseName => $files) {
        // ... (これまでのペアリングロジック) ...
    }

    return [
        'csv_data' => $csvFile ? ['name' => $csvFile->file_name, 'content' => $csvFile->content] : null,
        'obj_mtl_pairs' => $objMtlPairs
    ];
}
ステップ3：フロントエンドの修正 (objViewerStandard.js)

ここが最も大きな変更点です。fetchAndRenderModel関数を、CSVを先に読み込んでから、各OBJを配置するように全面的に書き換えます。

修正後のobjViewerStandard.jsの完全なコード

以下に、新しいアーキテクチャに対応したJavaScriptの完全なコードを示します。

code
JavaScript
download
content_copy
expand_less
// ... import文、グローバル変数、シーン設定などは、これまでの最終版と同じ ...

/**
 * 最終的なモデルデータをサーバーから取得し、3Dシーンを構築・描画する
 * 【最終修正版】CSVによる配置ロジック
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

        if (!response.csv_data || !response.obj_mtl_pairs) {
            throw new Error(`サーバーからのデータ形式が正しくありません。`);
        }
        
        const csvData = response.csv_data;
        const filePairs = response.obj_mtl_pairs;

        if (!csvData.content) {
            throw new Error(`全体配置座標を記載したCSVファイルが見つかりません。`);
        }
        if (!Array.isArray(filePairs) || filePairs.length === 0) {
            throw new Error(`表示できるOBJファイルがありません。`);
        }

        if (loaderTextElement) loaderTextElement.textContent = `モデルを構築しています...`;

        // =================================================================
        //  ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
        //
        //  **【重要】新しい組立ロジック**
        //

        // 2. CSVを解析し、部材IDをキーとする座標マップを作成
        const coordinatesMap = new Map();
        const csvLines = csvData.content.split(/\r?\n/);
        // ヘッダー行をスキップ (ヘッダーがなければ i = 0 から)
        for (let i = 1; i < csvLines.length; i++) {
            const line = csvLines[i].trim();
            if (line === '') continue;
            
            // スペースまたはカンマで列を分割
            const columns = line.split(/[ ,]+/); 
            
            const memberId = columns[0];
            const x = parseFloat(columns[1]);
            const y = parseFloat(columns[2]);
            const z = parseFloat(columns[3]);

            if (memberId && !isNaN(x) && !isNaN(y) && !isNaN(z)) {
                coordinatesMap.set(memberId, new THREE.Vector3(x, y, z));
            }
        }
        console.log("Coordinates Map created:", coordinatesMap);


        // 3. 各OBJ/MTLペアを解析し、シーンに追加
        const allLoadedObjects = [];
        // ... (ファイルペアをループして解析するロジックは、これまでのものと同じ) ...
        
        loadedObjectModelRoot = new THREE.Group();
        const validationBox = new THREE.Box3();

        allLoadedObjects.forEach(parsedGroup => {
            if (parsedGroup.isGroup) {
                while(parsedGroup.children.length > 0) {
                    const child = parsedGroup.children[0];

                    // 4. オブジェクトを配置する
                    // 部材IDをオブジェクト名から抽出（命名規則に合わせる必要あり）
                    // 例: オブジェクト名が "XXXX_some_details" の場合
                    const nameParts = child.name.split('_');
                    const memberId = nameParts[0];

                    const absolutePosition = coordinatesMap.get(memberId);

                    if (absolutePosition) {
                        // CSVに座標が見つかった場合、その位置に配置
                        child.position.copy(absolutePosition);
                    } else {
                        // 座標が見つからなかった場合の処理（原点に置くか、警告を出すなど）
                        console.warn(`Coordinates not found for member ID: ${memberId}. Placing at origin.`);
                    }

                    // 検証してルートオブジェクトに追加
                    if (child.isMesh && child.geometry) {
                        try {
                            validationBox.setFromObject(child);
                            loadedObjectModelRoot.add(child);
                        } catch (e) { /* ... */ }
                    }
                }
            }
        });
        
        if (loadedObjectModelRoot.children.length === 0) {
            throw new Error("有効なオブジェクトを読み込めませんでした。");
        }

        // 5. 【重要】モデル全体の中心移動は不要になる
        // const box = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        // const center = box.getCenter(new THREE.Vector3());
        // loadedObjectModelRoot.position.sub(center); // ← この行は削除またはコメントアウト
        
        // スケーリングと回転は、必要に応じて適用
        const scale = 0.01; // 例：mm単位のモデルをm単位に変換する場合
        loadedObjectModelRoot.scale.set(scale, scale, scale);
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;

        //
        //  ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
        // =================================================================
        
        // ... (カテゴリデータ取得、モデルツリー構築、シーンへの追加、カメラのフレーミングは同じ) ...
        
        await fetchAllCategoryData(parsedWSCenID, allIds);
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot); // 全て配置した後にカメラを調整
        updateInfoPanel();
        
    } catch (error) {
        // ...
    } finally {
        hideLoading();
    }
}

// ... これ以降のヘルパー関数は変更ありません ...
この修正による改善点

組立ロジックの実装:

fetchAndRenderModelは、まずCSVを解析してcoordinatesMapという座標の辞書を作成します。

次に、個別のOBJをロードした後、その部材IDをキーにしてcoordinatesMapから絶対座標を検索します。

child.position.copy(absolutePosition)を使って、各部品を正しい位置に配置します。

不要な処理の削除:

各部品が既に絶対座標に配置されるため、モデル全体を後から中央に移動させるloadedObjectModelRoot.position.sub(center)という処理は不要になります。これを削除しないと、座標がずれてしまうため注意が必要です。

柔軟なスケーリング:

CSVに記載されている座標の単位（mmなのかmなのか）に合わせて、scaleの値を調整することで、シーン全体の大きさを簡単に制御できます。

この修正により、課長様がご指示された、より高度で柔軟な「組立式」の3Dモデル表示アーキテクチャが実現されます。
