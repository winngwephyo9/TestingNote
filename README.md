function generatebukkenInfoByCenId(parsedWSCenID, displayId, infoPanel) {
    $.ajax({
        type: "post",
        url: url_prefix + "/DLDWH/getData",
        data: { _token: CSRF_TOKEN, WSCenID: parsedWSCenID, ElementId: displayId },
        success: function (data) {
            console.log("Get 「カテゴリ名」と「ファミリ名」");
            console.log(data);

            if (data && data.length > 0) {
                const categoryName = data[0]['カテゴリー名'];
                const familyName = data[0]['ファミリ名'];
                const typeId = data[0]['タイプ_ID'];
                const bukkenInfo = `\nカテゴリー名: ${categoryName}\nファミリ名: ${familyName}\nタイプ_ID: ${typeId}`;
                infoPanel.textContent += bukkenInfo;
            } else {
                infoPanel.textContent += '\nカテゴリー名: "" \nファミリ名: ""';
            }
        },
        error: function (err) {
            console.log(err);
            infoPanel.textContent += '\nデータ取得中にエラーが発生しました。';
        }
    });
}
