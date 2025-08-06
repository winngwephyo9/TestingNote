/* --- スタイルシートに追加・修正 --- */

/* 1. 親コンテナにposition: relative; を必ず指定します */
/* これがボタンを正しく配置するための最も重要な設定です */
#viewer-container {
    position: relative;
    /* widthやheightなど、既存のスタイルはそのままにしてください */
    width: 100%;
    height: 100%;
    overflow: hidden; /* コンテナからはみ出したものを隠す */
}

/* 2. 新しく作成したボタン用コンテナのスタイル */
/* このコンテナをビューワーの下部中央に配置します */
#viewer-controls-container {
    position: absolute; /* 親(viewer-container)を基準に配置 */
    bottom: 25px;       /* 下から25pxの位置 */
    left: 50%;          /* 左から50%の位置に移動 */
    transform: translateX(-50%); /* コンテナ自身の幅の半分だけ左に戻して中央揃え */
    z-index: 100;       /* canvasより手前に表示 */
    
    display: flex;      /* 中のボタンを横並びにする */
    align-items: center; /* ボタンの縦方向の位置を中央に揃える */
    gap: 12px;          /* ボタンとボタンの間の隙間を12pxに設定 */
}

/* 3. ボタンに共通のスタイルを適用 */
/* UI切り替えボタンのデザインを参考に、両方のボタンに統一感を持たせます */
#viewer-controls-container button {
    background-color: rgba(45, 52, 54, 0.8); /* 少し濃いグレー、半透明 */
    color: white;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
    font-weight: bold;
    font-size: 16px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 8px; /* 角を丸くする */
    padding: 10px 18px;
    cursor: pointer;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2); /* 影をつけて立体感を出す */
    transition: background-color 0.2s ease, transform 0.1s ease; /* アニメーション効果 */
}

/* マウスカーソルを合わせた時のスタイル */
#viewer-controls-container button:hover {
    background-color: rgba(9, 132, 227, 0.9); /* ホバー時に色を変更 */
}

/* クリックした（押した）時のスタイル */
#viewer-controls-container button:active {
    transform: translateY(1px); /* 少し下に沈むようなエフェクト */
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

/* ❌ボタンはアイコンなので、少しだけスタイルを微調整 */
#toggle-ui-button {
    padding: 10px 12px;
    font-size: 20px;
}

<!-- 3Dビューワーのメインコンテナ -->
<div id="viewer-container">
    
    <!-- (ここにthree.jsのcanvasが自動的に追加されます) -->

    <!-- NEW: ビューワー下部の中央に表示するコントロールボタン用のコンテナ -->
    <div id="viewer-controls-container">
        <!-- 「全体表示」ボタン -->
        <button id="resetViewButton" title="全体を表示（Reset View）">全体表示</button>
        
        <!-- 「UI表示/非表示」ボタン -->
        <button id="toggle-ui-button" title="UIパネルの表示/非表示">❌</button>
    </div>

</div>




