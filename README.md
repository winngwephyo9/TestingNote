<div id="viewer-container">
    <!-- このボタンを新しく追加 -->
    <button id="resetViewButton" title="全体を表示">全体表示</button> 

    <!-- 既存のUI切り替えボタン -->
    <button id="toggle-ui-button" title="Hide UI Panels">❌</button>
</div>



/* --- スタイルシートに追加 --- */

#resetViewButton {
    position: absolute;
    top: 10px; /* 上からの位置 */
    right: 50px; /* 右からの位置 (UIトグルボタンの隣あたり) */
    z-index: 10; /* canvasより手前に表示 */
    background-color: rgba(40, 40, 40, 0.7);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.5);
    border-radius: 4px;
    padding: 8px 12px;
    cursor: pointer;
    font-size: 14px;
}

#resetViewButton:hover {
    background-color: rgba(60, 60, 60, 0.9);
}

/* 既存のトグルボタンの位置を少し調整する必要があるかもしれません */
#toggle-ui-button {
    position: absolute;
    top: 10px;
    right: 10px;
    z-index: 10;
    /* ... 他のスタイル ... */
}


// --- Event Listeners and Animation Loop ---

// --- 変数定義 ---
let mouseDownPosition = new THREE.Vector2();

// --- NEW: リセットボタンの要素を取得 ---
const resetViewButton = document.getElementById('resetViewButton');


// --- イベントリスナー ---

// マウスのボタンが押された時の処理
viewerContainer.addEventListener('mousedown', (event) => {
    mouseDownPosition.set(event.clientX, event.clientY);
}, false);

// マウスのボタンが離された時の処理（クリックかドラッグかを判定）
viewerContainer.addEventListener('mouseup', (event) => {
    const mouseUpPosition = new THREE.Vector2(event.clientX, event.clientY);
    const distance = mouseDownPosition.distanceTo(mouseUpPosition);

    const DRAG_THRESHOLD = 5; 
    if (distance > DRAG_THRESHOLD) {
        return; // ドラッグ操作だったので、選択処理はしない
    }

    // 以下はクリックだった場合の選択処理
    if (!loadedObjectModelRoot) return;
    const rect = viewerContainer.getBoundingClientRect();
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);
    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let current = intersects[0].object;
        while (current && current.parent !== loadedObjectModelRoot && current !== loadedObjectModelRoot && current.parent !== scene) {
            current = current.parent;
        }
        if (current) newlyClickedTarget = current;
    }
    handleSelection(newlyClickedTarget);
}, false);


// --- NEW: リセットボタンがクリックされた時の処理 ---
if (resetViewButton) {
    resetViewButton.addEventListener('click', () => {
        // 1. オブジェクトの選択を解除する (これによりハイライトと隔離モードもリセットされる)
        handleSelection(null);

        // 2. カメラを、モデル全体が見える位置にリセットする
        if (loadedObjectModelRoot) {
            frameObject(loadedObjectModelRoot);
        }
    });
}


if (closeModelTreeBtn) closeModelTreeBtn.addEventListener('click', () => { if (modelTreePanel) modelTreePanel.style.display = 'none'; });

if (modelTreeSearch) {
    // ... (検索処理は変更なし) ...
}

if (toggleUiButton) {
    // ... (UIトグルボタンの処理も変更なし) ...
}

// ... (以降のコードは変更なし) ...
