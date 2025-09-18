/* ajax通信トークン定義 */
var CSRF_TOKEN = $('meta[name="csrf-token"]').attr('content');
var logContent = "";
var fileNames;
var folderNames;
var successMessage;

$(document).ready(function () {
    var login_user_id = $("#hidLoginID").val();
    var img_src = "/DL_DWH.png";
    var url = "DLDWHDataImport/index";
    var content_name = "情報データ取込";
    recordAccessHistory(login_user_id, img_src, url, content_name);
    showUpdateHistory();

});

/**
 * showUpdateHistory
 * 
 * 取込履歴の表示
 */
function showUpdateHistory() {
    var loginUserName = $("#hiddenLoginUser").val();
    $.ajax({
        type: "get",
        url: url_prefix + "/DLDWH/updateHistory",
        success: function (data) {
            if (data != null && data.length != 0) {
                var latestIndex = data.length - 1;
                var appendStr = "";
                var userName = data[latestIndex]["login_user"];
                var updating = data[latestIndex]["update_date"];
                var updateHistory = userName + " さんが " + updating + " に取り込みをしました。";

                appendStr += "<div class='dropdown'>";
                appendStr += "<div class='select'>";
                appendStr += "<i class='fa fa-chevron-left'></i>";
                appendStr += "<span>" + updateHistory + "</span>";
                appendStr += "</div>";
                appendStr += "<ul class='dropdown-menu'>";

                for (var i = latestIndex - 1; i >= 0; i--) {
                    var tmp_id = data[i]["id"];
                    var tmp_userName = data[i]["login_user"];
                    var tmp_updating = data[i]["update_date"];
                    var tmp_updateHistory = tmp_userName + " さんが " + tmp_updating + " に取り込みをしました。";

                    appendStr += "<li id='updateHistory" + tmp_id + "'>" + tmp_updateHistory + "</li>";
                }

                appendStr += "</ul>";
                appendStr += "</div>&nbsp";

                $('#update_history').empty();
                $('#update_history').append(appendStr);


            } else {
                var appendStr = "<span>取り込み履歴はありません。</span>";
                $('#update_history').empty();
                $('#update_history').append(appendStr);
            }

            /*Dropdown Menu*/
            $('.dropdown').click(function () {
                $(this).attr('tabindex', 1).focus();
                $(this).toggleClass('active');
                $(this).find('.dropdown-menu').slideToggle(300);
            });
            $('.dropdown').focusout(function () {
                $(this).removeClass('active');
                $(this).find('.dropdown-menu').slideUp(300);
            });

        },
        error: function (err) {
            console.log(err);
        }
    });
}

/**
 * importData
 * 
 * データベースにcsvデータを取り込む
 */
function importData() {
    var loginUserName = $("#hiddenLoginUser").val();
    // console.log(loginUserName);
    showLoading();
    $.ajax({
        type: "post",
        url: url_prefix + "/DLDWH/excel/import",
        data: { _token: CSRF_TOKEN, loginUserName: loginUserName },
        success: function (data) {
            console.log(data);
            if (data.token === "no_token") {
                hideLoading();
                alert("BOXにログインされていないため更新できませんでした。");
            } else {
                if (data.isfinishJob) {
                    console.log("finish job. no file upload.");
                    document.getElementById("result").innerHTML = "最新版のファイルがありません。";
                    hideLoading();
                } else {
                    console.log("The job is not yet finished.");
                    checkJobStatus();
                }
            }
        },
        error: function (err) {
            console.log(err);
            hideLoading();
            alert("データロードに失敗しました。\n管理者に問い合わせてください。");
        }
    });
}

/**
 * checkJobStatus
 * 
 * ジョブよりデータベースへの取込が完了したか50秒ごとに確認しに行く
 */
function checkJobStatus() {
    $.ajax({
        url: url_prefix + "/DLDWH/checkJobStatus",
        method: 'GET',
        success: function (response) {
            if (response.status === 'completed') {
                console.log(response.data);
                let data = response.data;
                fileNames = data.fileNames;
                folderNames = data.folderNames;
                successMessage = data.successMessage;
                // console.log(fileNames);
                // console.log(folderNames);
                // console.log(successMessage);

                const content = createLogContent(fileNames, successMessage);

                if (fileNames.length > 0) {
                    document.getElementById("result").innerHTML = content;
                } else {
                    document.getElementById("result").innerHTML = content;
                }

                showUpdateHistory();
                setTimeout(() => {
                    hideLoading();
                }, 1000);

            } else if (response.status === 'failedJob') {
                console.log("failedJob");
                console.log(response.data);
                document.getElementById("result").innerHTML = response.data;
                showUpdateHistory();
                setTimeout(() => {
                    hideLoading();
                }, 1000);
            }  else if(response.status === 'no_upload_file'){
                console.log("finish job. no file upload.");  
                document.getElementById("result").innerHTML = "最新版のファイルがありません。";              
                setTimeout(() => {
                    hideLoading();
                }, 1000);
            }else {
                // 一定時間後に再度チェック
                setTimeout(function () {
                    checkJobStatus();
                }, 5000); // 5000=5秒後に再チェック
            }
        }
    });
}

/**
 * createLogContent
 * 
 * ログの表示
 */
function createLogContent(fileNames, successMsg) {
    var content = "";
    var numErrors = 0;
    var numErrors = 0;
    if (successMsg.length > 0) {
        for (let i = 0; i < successMsg.length; i++) {
            for (const [key, values] of Object.entries(successMsg[i])) {
                // console.log("values " + values);
                // console.log("key " + key);
                var keyName = "";
                for (const value of values) {
                    if(keyName !== key){
                        // console.log("valueeeeee " + value);
                        for (const val of value){
                            if(val !== "OK"){
                                // console.log(value);
                                numErrors++;
                                break; 
                            }
                        }
                    }
                    keyName = key;
                }
            }
        }
        content = "ファイル" + successMsg.length + "個中" + numErrors + "個エラーがあります。";
    } else {
        content += "最新版のファイルがありません。";
    }
    return content;
}

/**
 * downloadLog
 * 
 * ログのダウンロード
 */
function downloadLog() {
    const today = new Date();
    const year = today.getFullYear();
    const month = String(today.getMonth() + 1).padStart(2, '0'); // Months are zero-based
    const day = String(today.getDate()).padStart(2, '0');
    const formattedDate = `${year}${month}${day}`;

    $.ajax({
        type: "post", // POSTメソッドを使用
        url: url_prefix + "/DLDWH/log/download",
        data: { fileNames: fileNames, folderNames: folderNames, successMessage: successMessage, formattedDate: formattedDate, _token: CSRF_TOKEN },
        xhrFields: {
            responseType: 'blob'
        },
        success: function (blob) {
            const url = window.URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.style.display = 'none';
            a.href = url;
            a.download = formattedDate + "_LOGS.xlsx"; // ファイル名を設定

            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            window.URL.revokeObjectURL(url);
        },
        error: function (err) {
            console.log(err);
            alert("ファイルのダウンロードに失敗しました。\n管理者に問い合わせてください。");
        }
    });
}

/**
 * deleteData
 * 
 * 全データ削除機能（仮）
 */
function deleteData() {
    showLoading();
    $.ajax({
        type: "post", // POSTメソッドを使用
        url: url_prefix + "/DLDWH/deleteData",
        data: { _token: CSRF_TOKEN },
        success: function (result) {
            console.log(result);
            showUpdateHistory();
            document.getElementById("result").innerHTML = "";
            setTimeout(() => {
                hideLoading();
            }, 1000);
        },
        error: function (err) {
            console.log(err);
            alert("データの削除が失敗しました。\n管理者に問い合わせてください。");
            hideLoading();
        }
    });
}

/**
 * ShowCompleteMessage
 * 
 * 取込完了警告メッセージ表示
 */
function ShowCompleteMessage(data) {
    toastr.success("取込完了しました。", "Success");
}


function showLoading() {
    $(".custom-loading").removeClass("hide");
    $(".custom-loading").addClass("show");
}


function hideLoading() {
    $(".custom-loading").removeClass("show");
    $(".custom-loading").addClass("hide");
}

