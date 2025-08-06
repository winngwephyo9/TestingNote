あと、セレクトリセットボタンみたいなのが必要かも…
何かを選択した時、絶対に手前に壁とか、他の要素があるので、
セレクトしてない状態に戻せないから、
どういう場所にいてるのかわかりにくいね…
<img width="1893" height="882" alt="image" src="https://github.com/user-attachments/assets/6c253e92-799e-4666-a35b-88ce5052bd29" />


// --- Event Listeners and Animation Loop ---

// REVISED: Listen for clicks only within the 3D viewer canvas.
// We will now distinguish between a "click" and a "drag" to prevent
// selecting objects after rotating the camera.

// --- 変数定義 ---
// マウスを押した位置を記憶するための変数
let mouseDownPosition = new THREE.Vector2();

// --- イベントリスナー ---
// マウスのボタンが押された時の処理
viewerContainer.addEventListener('mousedown', (event) => {
    // 押された瞬間のマウス座標を記録
    mouseDownPosition.set(event.clientX, event.clientY);
}, false);

// マウスのボタンが離された時の処理
viewerContainer.addEventListener('mouseup', (event) => {
    // ボタンが離された瞬間のマウス座標を取得
    const mouseUpPosition = new THREE.Vector2(event.clientX, event.clientY);
    
    // 押した位置と離した位置の距離を計算
    const distance = mouseDownPosition.distanceTo(mouseUpPosition);

    // この距離が非常に小さい場合（ほぼ動いていない場合）のみ「クリック」と判断する
    // この値（ピクセル数）は、必要に応じて調整してください
    const DRAG_THRESHOLD = 5; 
    if (distance > DRAG_THRESHOLD) {
        // 距離が大きい場合はドラッグ操作とみなし、選択処理を行わずに終了
        return;
    }

    // --- ここから下は、元々clickイベントにあった選択処理（レイキャスト） ---
    // 3Dモデルがロードされていなければ何もしない
    if (!loadedObjectModelRoot) return;

    // canvasの画面上の位置とサイズを取得
    const rect = viewerContainer.getBoundingClientRect();

    // マウスカーソルの位置を正規化デバイス座標（-1から1の範囲）に変換
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

    // レイキャスター（光線）をカメラの位置からマウスカーソルの方向へ設定
    raycaster.setFromCamera(mouse, camera);

    // 光線と交差したオブジェクトを全て取得
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);

    let newlyClickedTarget = null;
    // 交差したオブジェクトがあった場合
    if (intersects.length > 0) {
        // 一番手前にあるオブジェクトを取得
        let current = intersects[0].object;
        // そのオブジェクトの親をたどり、最上位のグループ（部品全体）を見つける
        while (current && current.parent !== loadedObjectModelRoot && current !== loadedObjectModelRoot && current.parent !== scene) {
            current = current.parent;
        }
        // 最上位のグループが見つかれば、それを選択ターゲットとする
        if (current) newlyClickedTarget = current;
    }

    // 見つかったターゲット（または何もなければnull）を選択処理関数に渡す
    handleSelection(newlyClickedTarget);

}, false);


if (closeModelTreeBtn) closeModelTreeBtn.addEventListener('click', () => { if (modelTreePanel) modelTreePanel.style.display = 'none'; });

// ... (以降のコードは変更ありません) ...
