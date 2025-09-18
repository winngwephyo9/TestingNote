 // 完了したのは現在選択中のモデルなので、それを再ロード
                const selectedOption = modelSelector.options[modelSelector.selectedIndex];
                initiateLoadProcess(selectedOption.value, selectedOption.text);
