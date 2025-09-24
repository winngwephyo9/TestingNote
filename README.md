<!-- 【新規】ロード中にメッセージを表示するための要素を追加 -->
    <span id="selector-message" style="display: none; margin-left: 10px; color: #ff6347;">
        ロード中は選択できません。
    </span>

    
    const selectorMessage = document.getElementById('selector-message');




    if (modelSelector) modelSelector.disabled = true; // ドロップダウンを無効化
    if (selectorMessage) selectorMessage.style.display = 'inline'; // メッセージを表示


if (modelSelector) modelSelector.disabled = false; // ドロップダウンを有効化
    if (selectorMessage) selectorMessage.style.display = 'none'; // メッセージを非表示
