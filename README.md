 if (error.responseJSON && error.responseJSON.error && error.responseJSON.error.includes('No cached model found')) {
            alert("表示できるデータがありません。BOXにログインしてデータを同期してください。");
            if (loaderTextElement) loaderTextElement.textContent = "表示できるデータがありません。";
        } else {
            if (loaderTextElement) loaderTextElement.textContent = `モデルの表示に失敗しました: ${error.message}`;
        }
