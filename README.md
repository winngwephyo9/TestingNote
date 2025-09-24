<img width="932" height="90" alt="image" src="https://github.com/user-attachments/assets/283acd00-bb85-46ae-88bf-90ff5f5dff1c" />
        // 未ログインの場合、メッセージを表示
            if (boxStatusMessage && boxStatusText) {
                boxStatusText.textContent =
                    'BOXにログインされていない場合、最新のデータを取得することができません。\n\nそのため、データベースに保存されている既存のモデルデータを表示いたします。<br>BOX側に更新されたデータが存在する場合は、最新の情報を取得するために、BOXへのログインをお願いいたします。';
                boxStatusMessage.style.display = 'flex'; // flexで表示
            }
