
            $mailAddressSearchAbsoluteLink = $addressBookBaseUrl . '/' . $mailAddressSearchLink;
            $finalResponse = $client->get($mailAddressSearchAbsoluteLink);
            $finalHtml = (string) $finalResponse->getBody();
            $finalHtml = mb_convert_encoding($finalHtml, 'UTF-8', 'SJIS-win');
            // dd('メールアドレス一覧の抽出 Page Content ', $finalHtml);


            //Auth
            $authAddressBookResponse = $client->get($addressSearchAuthUrl);
            $authAddressBookHtml = (string) $authAddressBookResponse->getBody();
            $authAddressBookHtml = mb_convert_encoding($authAddressBookHtml, 'UTF-8', 'SJIS-win');

  From above code,I got this html source code

<html>
<head>
  <meta http-equiv="Content-Type" content="text/html; charset=shift_jis" />
  <title>メールアドレス検索マニュアル</title>
<script>
function _submit() {
    if (location.href.indexOf("secure.fc") != -1) {
        f.action = "http://it-nw.isc.obayashi.co.jp/pick_new/frm/ADDR000.aspx";
    }
    f.submit();
}
function _close() {
    top.window.close();
}
</script>
<style>
  ol { margin-top:2;margin-bottom:2; }
  li { margin-top:2;margin-bottom:2; }
  a { color:blue; }
  a.chuui { color:red; }
</style>
</head>
<body style="margin-top:5;margin-bottom:0;">

<form method="POST" name="f" action="../frm/ADDR000.aspx" target="_top">
<input type="hidden" id="prev" name="prev" value="addr_man" />

<table cellspacing="0" style="width:100%;font-size:16;font-weight:bold;background-color:yellow;margin-top:0;margin-bottom:3;">
  <tr>
    <td>
      メールアドレス検索 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
      <input type="button" value="検索画面に進む" onClick="_submit()" /> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 
      <a href="faq.htm" target="naiyou">FAQ</a>
    </td>
    <td width="65" align="right">
      <input type="button" value="閉じる" onClick="_close()" />
    </td>
  </tr>
</table>

<table cellspacing="0" cellpadding="0" style="font-size:16;">
  <tr>
    <td width="258" valign="top">
      <ol>
        <li><a href="chuui.htm" target="naiyou" class="chuui">機能の概要と利用にあたっての注意事項 （必ずお読みください）</a></li>
        <li><a href="sousa.htm" target="naiyou">メールアドレス検索 操作方法</a> </li>
        <li><a href="seigen.htm" target="naiyou">検索時の制限について</a></li>
      </ol>
    </td>
    <td>
      <ol start="4">
        <li><a href="http://it.isc.obayashi.co.jp/tel_web/outlook_imp.html" target="naiyou">メールアドレスのOutlook連絡先へのインポート方法</a></li>
        <li><a href="issei.htm" target="naiyou">社内のたくさんの人に一度にまとめてメールを送信したい</a></li>
        <li><a href="hozon.htm" target="naiyou">検索条件の保存と読み込み</a></li>
      </ol>
    </td>
  </tr>
</table>

</form>
</body>
</html>

I want to go the next page when click 検索画面に進む button
the below html source code get from click 検索画面に進む




<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
<head>
  <title>メールアドレス検索</title>
<script type="text/javascript" language="javascript" src="../scripts/addr_common.js"></script>
<script type="text/javascript" language="javascript" src="../scripts/ADDR001.js"></script>
<style type="text/css">
  table { font-size:12px; }
  td { vertical-align:top; }
  /* td.title { background-color:#ccccff; } */
  td.koumoku { border-right:1px solid #000000;border-bottom:1px solid #000000;background-color:#ccccff; }
  td.koumoku2 { border-right:1px solid #000000;background-color:#ccccff; }
  td.sentaku { border-bottom:1px solid #000000; }
  a { color:#0000ff; }
  .col { color:#0000ff; }
  .col_sub { color:#000000; }
  .s1 { color:navy;font-size:13px; }
</style>
    <script language="javascript" type="text/javascript">
// <![CDATA[

        function chkTen_09_onclick() {

        }

// ]]>
    </script>
</head>
<body>


<form method="POST" id="form1" name="form1" action="ADDR101.aspx" target="gamen">
<div style="background-color:#ffff00;">
<table cellspacing="0" style="height:25px;">
  <tr><td style="font-size:16px;font-weight:bold;">メールアドレス・電話番号　検索</td></tr>
</table>
</div>
<div>
<table cellspacing="0" style="height:20px;">
  <tr><td style="font-size:12px;">絞り込みはAND検索です。</td></tr>
</table>
</div>

<table>
  <tr>
    <td><input type="button" id="btnNew" name="btnNew" value="新しい検索条件" onclick="btnNew_click();" /></td>
    <td style="text-align:right;"><input type="button" id="btnClose" name="btnClose" value="閉じる" onclick="btnClose_click();" /></td>
    <td></td>
  </tr>
  <tr>
    <td colspan="2"><input type="button" id="btnSave" name="btnSave" value="検索条件／出力項目　の保存" onclick="btnSave_click();" /></td>
    <td><input type="button" id="btnLoad" name="btnLoad" value="検索条件／出力項目　の読込" onclick="btnLoad_click();" /></td>
  </tr>
</table>

<div style="height:700px;">
<table style="width:97.8%">
  <tr>
    <td style="width:50%;vertical-align:top;">
      <table cellspacing="0" style="border:1px solid #000000;width:100%;">
        <colgroup>
          <col style="width:50%" />
          <col style="width:50%" />
        </colgroup>
        <tr>
          <th style="border-right:1px solid #000000;border-bottom:1px solid #000000;color:#008080;">条件</th>
          <th style="border-bottom:1px solid #000000;color:#008080;">条件の内容</th>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">支店</span><br />
            <span class="col_sub">・未選択の場合は全店が対象</span>
          </td>
          <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td><input type="checkbox" id="chkTen_09" name="chkTen_09" value="09"  /><label for="chkTen_09">本社</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_11" name="chkTen_11" value="11"  /><label for="chkTen_11">東京本店・関東支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_15" name="chkTen_15" value="15"  /><label for="chkTen_15">横浜支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_10" name="chkTen_10" value="10"  /><label for="chkTen_10">大阪本店・京都支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_26" name="chkTen_26" value="26"  /><label for="chkTen_26">神戸支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_12" name="chkTen_12" value="12"  /><label for="chkTen_12">名古屋支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_13" name="chkTen_13" value="13"  /><label for="chkTen_13">九州支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_14" name="chkTen_14" value="14"  /><label for="chkTen_14">東北支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_16" name="chkTen_16" value="16"  /><label for="chkTen_16">札幌支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_17" name="chkTen_17" value="17"  /><label for="chkTen_17">広島支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_19" name="chkTen_19" value="19"  /><label for="chkTen_19">四国支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_27" name="chkTen_27" value="27"  /><label for="chkTen_27">北陸支店</label></td></tr>
              <!--<tr><td><input type="checkbox" id="chkTen_30" name="chkTen_30" value="30"  /><label for="chkTen_30">海外支店</label></td></tr>-->
              <tr><td><input type="checkbox" id="chkTen_31" name="chkTen_31" value="31"  /><label for="chkTen_31">アジア支店</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_32" name="chkTen_32" value="32"  /><label for="chkTen_32">北米支店</label><td></tr>
              <tr><td><input type="checkbox" id="chkTen_46" name="chkTen_46" value="46"  /><label for="chkTen_46">東日本ロボティクスセンター</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_36" name="chkTen_36" value="36"  /><label for="chkTen_36">西日本ロボティクスセンター</label></td></tr>
              <tr><td><input type="checkbox" id="chkTen_66" name="chkTen_66" value="66"  /><label for="chkTen_66">出向</label></td></tr>
              <tr>
                <td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.Ten);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td>
              </tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">組織コード</span><br />
            <span class="col_sub">・未入力の場合は全組織が対象</span><br />
            <span class="col_sub">・半角数字で5文字または7文字</span><br />
            <span class="col_sub">・前方一致</span><br />
            <span class="col_sub">・複数の組織コードを入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">&nbsp;&nbsp;&nbsp;&nbsp;例：09117,0911222</span><br />
            <span class="col_sub">・1000文字以内</span>
          </td>
          <td class="sentaku">
            <textarea id="txtSosikiCode" name="txtSosikiCode" rows="2" cols="20" style="height:100px;width:95%;overflow:hidden;ime-mode:disabled;"></textarea>
          </td>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">組織名</span><br />
            <span class="col_sub">・未入力の場合は全組織が対象</span><br />
            <span style="color:#ff0000;">・設定すると、検索時間が長くなるため、なるべく設定しない</span><br />
            <span class="col_sub">・全角</span><br />
            <span class="col_sub">・部分一致</span><br />
            <span class="col_sub">・複数の組織名を入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">&nbsp;&nbsp;&nbsp;&nbsp;例：本社総務部,グローバル</span><br />
            <span class="col_sub">・3000文字以内</span>
          </td>
          <td class="sentaku">
            <textarea id="txtSosikiName" name="txtSosikiName" rows="2" cols="20" style="height:120px;width:95%;overflow:hidden;ime-mode:active;"></textarea>
          </td>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">常設・現場</span><br />
            <span class="col_sub">・未選択の場合は全員が対象</span>
          </td>
          <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td>
                <input type="checkbox" id="chkJG_001" name="chkJG_001"  value="1" /><label for="chkJG_001">常設</label>
                <input type="checkbox" id="chkJG_002" name="chkJG_002"  value="1" /><label for="chkJG_002">現場</label>
              </td></tr>
              <tr>
                <td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.JGKubun);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td>
              </tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="koumoku2">
            <span class="col">工事区分</span><br />
            <span class="col_sub">・未選択の場合は全工事区分が対象</span>
          </td>
          <td>
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td>
                <input type="checkbox" id="chkKKubun_001" name="chkKKubun_001" value="1"  /><label for="chkKKubun_001">国内土木</label>
                <input type="checkbox" id="chkKKubun_002" name="chkKKubun_002" value="1"  /><label for="chkKKubun_002">海外土木</label>
              </td></tr>
              <tr><td>
                <input type="checkbox" id="chkKKubun_003" name="chkKKubun_003" value="1"  /><label for="chkKKubun_003">国内建築</label>
                <input type="checkbox" id="chkKKubun_004" name="chkKKubun_004" value="1"  /><label for="chkKKubun_004">海外建築</label>
              </td></tr>
              <tr>
                <td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.KKubun);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td>
              </tr>
            </table>
          </td>
        </tr>
      </table>
    </td>
    <td style="width:10px;"></td>
    <td>
      <table cellspacing="0" style="border:1px solid #000000;width:100%;">
        <colgroup>
          <col style="width:50%" />
          <col style="width:50%" />
        </colgroup>
        <tr>
          <th style="border-right:1px solid #000000;border-bottom:1px solid #000000;color:#008080;">条件</th>
          <th style="border-bottom:1px solid #000000;color:#008080;">条件の内容</th>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">個人コード</span><br />
            <span class="col_sub">・未入力の場合は全員が対象</span><br />
            <span class="col_sub">・前方一致</span><br />
            <span class="col_sub">・複数の個人コードを入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">&nbsp;&nbsp;&nbsp;&nbsp;例：11111,21</span><br />
            <span class="col_sub">・1000文字以内</span>
          </td>
          <td class="sentaku">
            <textarea id="txtKojinCode" name="txtKojinCode" rows="2" cols="20" style="height:110px;width:95%;overflow:hidden;ime-mode:disabled;" maxlength="100"></textarea>
          </td>
        </tr>
        <tr>
          <td rowspan="2" class="koumoku">
            <span class="col">協力スタッフ</span><br />
            <span class="col_sub">・未選択の場合は全員が対象</span><br />
            <span class="col_sub">・個人コード1文字目</span><br />
            <span class="col_sub">・複数の文字を入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">・例：H,S,M,J,G,X,Y,Z</span><br />
            <span class="col_sub">・50文字以内</span><br />
            「個人コード体系については<a href="http://it.isc.obayashi.co.jp/security/mng_staff/userid.htm" target="_blank">こちら</a>」
          </td>
          <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td><span class="s1">以下の個人コードを</span><br /></td></tr>
              <tr><td>
                <input type="radio" id="rdoKyoryoku01" name="rdoKyoryoku" value="1"  /><label for="rdoKyoryoku01">抽出する</label>
                <input type="radio" id="rdoKyoryoku02" name="rdoKyoryoku" value="2"  /><label for="rdoKyoryoku02">除外する</label>
              </td></tr>
              <tr><td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.Kyoryoku);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td></tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="sentaku">
            <textarea id="txtKyoryoku" name="txtKyoryoku" rows="2" cols="20" style="height:50px;width:95%;overflow:hidden;ime-mode:disabled;"></textarea>
          </td>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">本務・兼務・他在籍・仮想在籍</span><br />
            <span class="col_sub">・未選択の場合は全員が対象</span>
          </td>
          <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td>
                <input type="checkbox" id="chkHonKen_001" name="chkHonKen_001" value="1"  /><label for="chkHonKen_001">本務</label>
                <input type="checkbox" id="chkHonKen_002" name="chkHonKen_002" value="1"  /><label for="chkHonKen_002">兼務</label>
                <input type="checkbox" id="chkHonKen_003" name="chkHonKen_003" value="1"  /><label for="chkHonKen_003">他在籍</label>
              </td></tr>
              <tr><td><input type="checkbox" id="chkHonKen_004" name="chkHonKen_004" value="1"  /><label for="chkHonKen_004">仮想在籍</label></td></tr>
              <tr><td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.HonKen);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td></tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="koumoku">
            <span class="col">職位（セキュリティランク）</span><br />
            <span class="col_sub">・未選択の場合は全職位が対象</span><br />
            「職位については<a href="http://it.isc.obayashi.co.jp/ct_sys/secure_web/secrank.htm" target="_blank">こちら</a>」<br />
            <span class="col_sub">・本務・兼務・他在籍・仮想在籍の条件が未選択の場合、兼務・他在籍・仮想在籍の職位情報も検索範囲になります。</span>
          </td>
          <td class="sentaku">
            <select id="ddtSecRankS" name="ddtSecRankS" style="width:160px;">
            <option value="" selected ></option>
<option value="1">役員・理事</option>
<option value="3">部長・所長</option>
<option value="4">副部長・課長</option>
<option value="5">副課長</option>
<option value="7">主任・職員</option>
<option value="8">期間職員・期間雇用</option>
<option value="11">ｽﾀｯﾌ(従業員同等)</option>
<option value="13">ｽﾀｯﾌ(業務ｼｽﾃﾑ利用)</option>
<option value="14">ｽﾀｯﾌ(電話帳・ｱﾄﾞﾚｽ帳)</option>
<option value="15">ｽﾀｯﾌ(電子ﾒｰﾙのみ)</option>

            </select>～<br />
            <select id="ddtSecRankE" name="ddtSecRankE" style="width:160px;">
            <option value="" selected ></option>
<option value="1">役員・理事</option>
<option value="3">部長・所長</option>
<option value="4">副部長・課長</option>
<option value="5">副課長</option>
<option value="7">主任・職員</option>
<option value="8">期間職員・期間雇用</option>
<option value="11">ｽﾀｯﾌ(従業員同等)</option>
<option value="13">ｽﾀｯﾌ(業務ｼｽﾃﾑ利用)</option>
<option value="14">ｽﾀｯﾌ(電話帳・ｱﾄﾞﾚｽ帳)</option>
<option value="15">ｽﾀｯﾌ(電子ﾒｰﾙのみ)</option>

            </select>
          </td>
        </tr>
        <tr>
          <td rowspan="2" class="koumoku">
            <span class="col">役職</span><br />
            <span class="col_sub">・未選択の場合は全役職が対象</span><br />
            <span class="col_sub">・部分一致</span><br />
            <span class="col_sub">・複数の役職を入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">&nbsp;&nbsp;&nbsp;&nbsp;例：部長,課長</span><br />
            <span class="col_sub">・300文字以内</span>
            </td>
            <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td><span class="s1">以下の役職を</span></td></tr>
              <tr><td>
                <input type="radio" id="rdoYaku01" name="rdoYaku" value="1"  /><label for="rdoYaku01">抽出する</label>
                <input type="radio" id="rdoYaku02" name="rdoYaku" value="2"  /><label for="rdoYaku02">除外する</label>
              </td></tr>
              <tr><td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.Yaku);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td></tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="sentaku">
            <textarea id="txtYaku" name="txtYaku" rows="2" cols="20" style="height:50px;width:95%;overflow:hidden;ime-mode:active;"></textarea>
          </td>
        </tr>
        <tr>
          <td rowspan="2" class="koumoku">
            <span class="col">従区分名</span><br />
            <span class="col_sub">・未選択の場合は全従区分が対象</span><br />
            <span class="col_sub">・部分一致</span><br />
            <span class="col_sub">・複数の従区分名を入力する場合は、半角「,」で区切る</span><br />
            <span class="col_sub">&nbsp;&nbsp;&nbsp;&nbsp;例：職員,役員</span><br />
            <span class="col_sub">・300文字以内</span>
          </td>
          <td class="sentaku">
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td><span class="s1">以下の従区分名を</span></td></tr>
              <tr><td>
                <input type="radio" id="rdoJuKubun01" name="rdoJuKubun" value="1"  /><label for="rdoJuKubun01">抽出する</label>
                <input type="radio" id="rdoJuKubun02" name="rdoJuKubun" value="2"  /><label for="rdoJuKubun02">除外する</label>
              </td></tr>
              <tr>
                <td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.JuKubun);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td>
              </tr>
            </table>
          </td>
        </tr>
        <tr>
          <td class="sentaku">
            <textarea name="txtJuKubun" rows="2" cols="20" id="txtJuKubun" style="height:50px;width:95%;overflow:hidden;ime-mode:active;"></textarea>
          </td>
        </tr>
        <tr>
          <td class="koumoku2">
            <span class="col">職種</span><br />
            <span class="col_sub">・未選択の場合は全職種が対象</span>
          </td>
          <td>
            <table cellpadding="0" cellspacing="0" style="width:100%;">
              <tr><td>
                <input type="checkbox" id="chkSyoku_001" name="chkSyoku_001" value="1"  /><label for="chkSyoku_001">建築</label>
                <input type="checkbox" id="chkSyoku_002" name="chkSyoku_002" value="1"  /><label for="chkSyoku_002">土木</label>
                <input type="checkbox" id="chkSyoku_003" name="chkSyoku_003" value="1"  /><label for="chkSyoku_003">事務</label>
              </td></tr>
              <tr><td>
                <input type="checkbox" id="chkSyoku_004" name="chkSyoku_004" value="1"  /><label for="chkSyoku_004">設備</label>
                <input type="checkbox" id="chkSyoku_005" name="chkSyoku_005" value="1"  /><label for="chkSyoku_005">機電</label>
                <input type="checkbox" id="chkSyoku_006" name="chkSyoku_006" value="1"  /><label for="chkSyoku_006">その他</label>
              </td></tr>
              <tr>
                <td style="text-align:right;"><a href="javascript:void(0);" onclick="return onClear(enumKoumoku.Syoku);">クリア</a>&nbsp;&nbsp;&nbsp;&nbsp;</td>
              </tr>
            </table>
          </td>
        </tr>
      </table>
    </td>
  </tr>
</table>
</div>

<input type="hidden" id="hidKojinCode" name="hidKojinCode" value="" />
<input type="hidden" id="hidHonken" name="hidHonken" value="" />
<input type="hidden" id="hidKojinName" name="hidKojinName" value="" />
<input type="hidden" id="hidKojinKana" name="hidKojinKana" value="" />
<input type="hidden" id="hidRank" name="hidRank" value="" />
<input type="hidden" id="hidMailAddress" name="hidMailAddress" value="" />
<input type="hidden" id="hidYaku" name="hidYaku" value="" />
<input type="hidden" id="hidTenCode" name="hidTenCode" value="" />
<input type="hidden" id="hidJukubun" name="hidJukubun" value="" />
<input type="hidden" id="hidTenMei" name="hidTenMei" value="" />
<input type="hidden" id="hidJukubunName" name="hidJukubunName" value="" />
<input type="hidden" id="hidSosikiCode" name="hidSosikiCode" value="" />
<input type="hidden" id="hidSyoku" name="hidSyoku" value="" />
<input type="hidden" id="hidSosikiMei" name="hidSosikiMei" value="" />
<input type="hidden" id="hidNaisen" name="hidNaisen" value="" />
<input type="hidden" id="hidJGKubun" name="hidJGKubun" value="" />
<input type="hidden" id="hidGaisen" name="hidGaisen" value="" />
<input type="hidden" id="hidKoujiKubun" name="hidKoujiKubun" value="" />
<input type="hidden" id="hidWorksite" name="hidWorksite" value="" />
</form>


<form method="POST" id="form2" name="form2" action="./ADDR201.aspx" target="_top">
<input type="hidden" id="txtDummy" name="txtDummy" value="ADDR001" />
</form>

</body>
</html>

I want to get the above html source, So modify code






^ "An error occurred: cURL error 6: Could not resolve host: address_search.html (see https://curl.haxx.se/libcurl/c/libcurl-errors.html)"

<html>

<head>
<meta http-equiv="Content-Type" content="text/html; charset=x-sjis">
<!--<script src="http://www.fc.obayashi.co.jp/accessCookie.js"></script>-->
<SCRIPT LANGUAGE="JavaScript">
    var n;
    n = getCookie("secCode");
    if ( n!= "" && n > 13 ) {
	window.location = "http://www.fc.obayashi.co.jp/deny.htm";

    }
</SCRIPT>
<title>メールアドレス一覧の抽出</title>
</head>


<body link="#0000FF" vlink="#0000FF" alink="#FF0000">
  <a href="index.html" target="_top"><span style="font-size:smaller;">電話帳・アドレス帳へ戻る</span></a>


  <table border="0" cellspacing="0" id="TABLE1" width="100%">
    <tr>
      <td height="2" width="755" bgcolor="#FFFF66"><span style="font-size:medium; font:bold; color:#800080;">
        メールアドレス一覧の抽出</span></td>
    </tr>
  </table>


  <table border="0" cellspacing="0" id="TABLE1" width="100%">
    <tr>
      <td align="right" bgcolor="#FFCC00">
        <p align="right"><span style="font-size:smaller; color:#800080;">メールアドレス一覧を取得します。</span></p></td>
    </tr>
  </table>

  <hr>

  <p align="left">
  　メールアドレスを抽出するツールとして、以下のツールが用意されています。<br>
  　このツールで抽出したメールアドレスをOutlookの連絡先に取り込むことで、一斉メールの発信が容易になります。  
　</p>


  <blockquote>
    <div align="center">
      <table border="1" cellspacing="0" width="93%" cellpadding="3" bordercolorlight="#FFFFFF" bordercolordark="#000000">
        <tr>
          <td width="20%" bgcolor="#B7CACA" bordercolor="#FFFFFF" align="left" height="160" bordercolorlight="#FFFFFF" bordercolordark="#C0C0C0">
            <span style="font-size:medium; color:#FFFF66;">■</span><span style="font-size:medium; font:bold;"><a target="_blank" href="http://it-nw.isc.obayashi.co.jp/pick/manual/addr_man.htm">メールアドレス検索システム</a></span><br /><br />
            <!--<span style="color:#F0F; font-weight:bold; font-size:14px; margin-left:10px;">8/13～8/15の期間は、</span><br /><span style="color:#F0F; font-weight:bold; font-size:14px; margin-left:10px;">サーバメンテナンス中のため</span><br />
            <span style="color:#F0F; font-weight:bold; font-size:14px; margin-left:10px;">ご利用いただけません。</span>--></td>
          <td width="4%" bordercolor="#FFFFFF" bgcolor="#FFFFCC" height="160" bordercolorlight="#FFFFFF" bordercolordark="#C0C0C0" align="center">
            <a><span style="font-size:smaller;">詳細</span></a></td>
          <td width="41%" bordercolor="#FFFFFF" height="160" bordercolorlight="#FFFFFF" bordercolordark="#C0C0C0">
            <span style="font-size:smaller;">　所属等を指定して該当者一覧のメールアドレスや氏名等を抽出することができます。<br>
            　メールを送付する対象者が、店をまたがっていたりする場合や店内全員など、人数が多いときに便利です。<br></span>
            <span style="font-size:smaller; color:#FF0000;">
            　多種多様に検索を行いたい場合は、多めに検索し、検索結果のＣＳＶファイルをパソコンにダウンロードした後、<br>
            　ＥＸＣＥＬのオートフィルタ機能や手作業で、必要なデータのみにしてください。<br></span>
	        <span style="font-size:smaller; color:#FF0000;">
	        　作成した一覧データは、目的を達成し次第削除してください。また他の人にコピーを渡すことは行わないでください。
	        </span></td>
        </tr>
      </table>
    </div>
  </blockquote>


  <p align="left">
<span style="font:bold;">　　　　※Outlookの連絡先へアドレスデータを取り込む方法は<a target="_blank" href="http://it.isc.obayashi.co.jp/tel_web/outlook_imp.html">こちら</a>をご覧ください。<br><br><br><br>
      <!--<span style="color:#FFFF66;">■</span><span style="font-size:medium;"><a href="http://it-nw.isc.obayashi.co.jp/pick/manual/addr_man.htm" name="001">メールアドレス検索システム</a></span>
    </span>
    <i><span style="color:#FF0000;">(2004年6月より公開しました)</span></i>　<span style="font-size:smaller;">（左のメニューをクリックしてください）</span>
  </p>
-->

<blockquote>
    <div align="center">
      <table border="1" cellspacing="0" width="93%" cellpadding="5" bgcolor="#FFFFFF">
        <tr>
          <td width="17%" bgcolor="#B7CACA" bordercolor="#FFFFFF" align="center">機能</td>
          <td width="53%" bordercolor="#FFFFFF" bgcolor="#CCCCCC">
          <span style="font-size:smaller;">　所属等を指定して該当者一覧のメールアドレスや氏名等を抽出することができます。<br>
          　抽出したデータは、Outlookの連絡先に取り込んだり、Ｗｅｂアンケートシステム等に利用できます。</span></td>
        </tr>
        <tr>
          <td width="17%" bgcolor="#B7CACA" bordercolor="#FFFFFF" align="center">目的</td>
          <td width="53%" bordercolor="#FFFFFF" bgcolor="#CCCCCC">
            <span style="font-size:smaller;">　従業員への一斉メール、Ｗｅｂアンケートシステムの実施などに利用することを目的としています。</span></td>
        </tr>
        <tr>
          <td width="17%" bgcolor="#B7CACA" bordercolor="#FFFFFF" align="center">アクセス制限</td>
          <td width="53%" bordercolor="#FFFFFF" bgcolor="#CCCCCC">
            <span style="font-size:smaller; color:#FF0000;">　個人情報の漏洩防止、メールの誤送信防止の観点から、本システムは特定の部署にのみアクセス権を設定します。</span><br>
            <span style="font-size:smaller;">
            　許可されている部署の方は、上記メニューをクリックすると同システムに入ることができます。一方、許可されていない部署の方は、<br>
            　上記メニューをクリックすると「アクセスする権限がありません」と表示されます。
            　（参考）本システムは<a target="_blank" href="http://it.isc.obayashi.co.jp/ct_sys/secure_web/">アクセス制限付き共用Webサーバ</a>の機能を利用しています。</span></td>
        </tr>
        <tr>
          <td width="17%" bgcolor="#B7CACA" bordercolor="#FFFFFF" align="center">アクセス許可の依頼</td>
          <td width="53%" bordercolor="#FFFFFF" bgcolor="#CCCCCC"><span style="font-size:smaller;">　業務上このシステムの利用が必要な部門の方は、利用目的を明記し、<a href="mailto:mail_question@mb.obayashi.co.jp?subject=メールアドレス検索システム利用申請">メールアドレス検索システム　担当者</a>までメールにてご依頼ください。依頼いただく際には、CCに所属長を含めてご連絡ください。<br>
            　アクセス権を設定します。設定完了後連絡いたします。なお、設定から連絡まで数日かかりますので、利用される際には、ゆとりを持ってお申し込みください。</span></td>
        </tr>
        <tr>
          <td width="17%" bgcolor="#B7CACA" bordercolor="#FFFFFF" align="center">利用時間の制限</td>
          <td width="53%" bordercolor="#FFFFFF" bgcolor="#CCCCCC">
          <span style="font-size:smaller;">　抽出はサーバに負荷を与えるため、始業時等一部時間帯は利用制限を設けさせていただきます。</span></td>
        </tr>
        <tr>
          <td width="17%" bgcolor="#B7CACA" bordercolor="#FFFFFF" align="center">利用権限の棚卸</td>
          <td width="53%" bordercolor="#FFFFFF" bgcolor="#CCCCCC">
          <span style="font-size:smaller;">
          　年1回、利用権限の棚卸を実施致します。<br>
          　通知が来ましたら、利用の有無と利用目的をご回答ください。</span></td>
        </tr>
      </table>  
    </div>
  </blockquote>

  <hr>
  
  <p align="center">
    <span style="font-size:smaller;">Copyright&copy; 2002-2015 OBAYASHI Corporation All Rights Reserved. </span>
  </p>
</body>
</html>


// 3. frame 要素から src 属性の値を取得
            $frameSrc = $addressBookIndexCrawler->filter('frame#frame1')->attr('src');

            if (!$frameSrc) {
                dd('Frame source URL not found.');
            }

            // 絶対URLに変換
            $addressBookFrameUrl = $addressBookBaseUrl . '/' . ltrim($frameSrc, './');

            // 4. frame の内容 (ob_index.aspx) を取得
            $addressBookFrameResponse = $client->get($addressBookFrameUrl);
            $addressBookFrameHtml = (string) $addressBookFrameResponse->getBody();
            $addressBookFrameCrawler = new Crawler($addressBookFrameHtml);

            // 5. 「メールアドレス検索システム」リンクの抽出
            $mailAddressSearchLinkNode = $addressBookFrameCrawler->filter('a:contains("メールアドレス検索システム")');

            if ($mailAddressSearchLinkNode->count() > 0) {
                $mailAddressSearchLink = $mailAddressSearchLinkNode->attr('href');
            } else {
                dd('メールアドレス検索システム link not found in the 電話帳 frame.');
            }
	    
     dd('電話帳へのアクセス!', $addressBookCrawler);
<?xml version="1.0" standalone="yes"?>
      <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.0 Transitional//EN" "http://www.w3.org/TR/REC-html40/loose.dtd">
      <?xml encoding="UTF-8"?>
      <html>
        <body><p>&#xFEFF;&#xD;
      </p>&#xD;
        <title>&#x96FB;&#x8A71;&#x5E33;&#x30FB;&#x30A2;&#x30C9;&#x30EC;&#x30B9;&#x5E33;</title>&#xD;
      &#xD;
      <frameset rows="100%" frameborder="0">&#xD;
      <frame id="frame1" name="frame1" src="./ob_index.aspx"/>&#xD;
      <noframes>&#xD;
      <div>&#x3053;&#x306E;&#x30DA;&#x30FC;&#x30B8;&#x3092;&#x3054;&#x89A7;&#x3044;&#x305F;&#x3060;&#x304F;&#x306B;&#x306F;&#x30D5;&#x30EC;&#x30FC;&#x30E0;&#x5BFE;&#x ▶
      </noframes>&#xD;
      </frameset>&#xD;
      &#xD;
      </body>
      </html>
      
      
      <div id="sidemenu"><a href="http://www.obayashi.co.jp/" target="_blank" id="to_obayashiHP"><img src="images/obayashiweb.gif" alt="社外ホームページへ" width="170" height="38"></a>
  <div id="h2_box">
<h2>サイドメニュー</h2>
<a href="javascript:;" onclick="blind_toggle.start(); return false;">設定</a>
<br class="clearfloat">
</div>

<div id="slide_setmenu" style="overflow: hidden; visibility: visible; display: none;">
<div class="sp01">
サイドメニューの展開状態を<br>
保存しますか？
</div>
<form name="form">
<div class="sp02">
  <input type="radio" name="set_sidemenu" value="keep" id="set_sidemenu_0"><span>保持する</span>
  <br>
  <input type="radio" name="set_sidemenu" value="close" id="set_sidemenu_1"><span>起動時はいつも閉じる</span>
  <br>
  <input type="radio" name="set_sidemenu" value="open" id="set_sidemenu_2"><span>起動時はいつも開く</span>
</div>
<div class="sp03">
	<input name="set" type="button" value="設定する" onclick="CookieCheck(),blind_toggle.start(); return false;">
	<input name="set" type="button" value="中止" onclick="blind_toggle.start(); return false;">
</div>
</form>
</div>

<ul id="list01" class="ul01">
<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.csr.obayashi.co.jp/topics/assets_images/kihon_rinen_2018rev.pdf" target="_blank" class="first_a">基本理念</a></li>

<li class="li_normal_sec"><img src="images/bulet_side.gif" class="bulet"><a href="http://secure.fc.obayashi.co.jp/accdept/sougoukikaku/log/clicklog.asp?url=http://secure.fc.obayashi.co.jp/accdept/sougoukikaku/overdirectorandauditor/index.htm&amp;title=keiei" target="_blank" class="first_a">経営情報</a></li>


<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.brand.obayashi.co.jp/pdf/DL_slogan_A4.pdf" target="_blank" class="first_a">ブランドビジョン</a></li>

<li class="li_long"><strong><font size="1"><img src="images/bulet_side.gif" class="bulet"><a href="http://gdi.obayashi.co.jp/chukei2022/" target="_blank" class="first_a">中期経営計画２０２２ポータル</a></font></strong></li>
<li class="li_long_5"><a href="http://secure.fc.obayashi.co.jp/accdept/sougoukikaku/newhomepage/03%20keieikeikaku/2025sisaku.pdf" class="rinri_4" target="_blank">２０２５年度経営施策</a></li>

<li class="li_long_2"><img src="images/bulet_side.gif" class="bulet"><a href="http://gdi.obayashi.co.jp/seisansei/mtg.html" target="_blank" class="first_c">会議改革ポータル</a></li>

<br>

<li class="li_long_2"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.ga.obayashi.co.jp/tekiseika/kigyourinri/kigyourinri.htm" target="_blank" class="first_c">企業倫理</a></li>
<li class="li_long_5"><a href="http://www.ga.obayashi.co.jp/rinritsuho/tsuho.pdf" class="rinri_4" target="_blank">企業倫理相談・通報制度</a></li>

<li class="li_long_5"><a href="http://www.ga.obayashi.co.jp/tekiseika/kigyourinri/kaigoukonsinrepo.html" class="rinri_4" target="_blank">同業者会合・懇親会報告</a></li>

<li class="li_long_5"><a href="http://www.ga.obayashi.co.jp/tekiseika/kigyourinri/training/portal/top.html" class="rinri_4" target="_blank">職場内研修ポータル</a></li>

<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://gdi.obayashi.co.jp/" target="_blank" class="first_digital">ダイバーシティ＆インクルージョン</a></li>

<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.ga.obayashi.co.jp/kikikanri/kikikanri.htm" target="_blank" class="first_a">危機管理</a></li>
	<li class="li_normal" style="margin-right:-3px;"><img src="images/bulet_side.gif" class="bulet"><span style="font-size:95%"><a href="http://www2.ga.obayashi.co.jp/e-portal/index.html" target="_blank" class="second_a">災害対策本部</a></span></li>
<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/harassment" target="_blank" class="first_a">ハラスメント</a></li>
<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><span style="font-size:95%"><a href="http://www.ga.obayashi.co.jp/rinritsuho/soudan.html" target="_blank" class="first_a">各種相談窓口</a></span></li>
</ul>
<br class="clearfloat">

<div id="CollapsiblePanel1" class="CollapsiblePanel clearfix CollapsiblePanelClosed">

  <div class="CollapsiblePanelTab" tabindex="1">
  <h3><img src="images/side_h3_01.gif" alt="ポータル" onclick="tabwrite1()"></h3>
  </div>
  <div class="CollapsiblePanelContent2" style="display: none;">
	<ul class="ul01">
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.pd.obayashi.co.jp/jinji/jinjiportal/" target="_blank" class="first_a">人事給与</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.pd.obayashi.co.jp/jinji/work_innovation/" target="_blank" class="first_a">働き方改革</a></li>
		<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.pd.obayashi.co.jp/jinji/recruit/" target="_blank" class="first_a">学生の社員訪問申請</a></li>
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.dho.cvl.obayashi.co.jp/c-portal/" target="_blank" class="first_a">土木</a><a href="javascript:;" class="ichiniti" onclick="javascript:window.open('http://www.dho.cvl.obayashi.co.jp/new_ichinichi_ichiwa/','cvlskk','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,top=30,left=400');"><img src="images/civil.gif" alt="土木一日一話" width="65" height="11"></a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.dho2.bc.obayashi.co.jp/" target="_blank" class="first_a">建築</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/eigyo_portal/SitePages/Home.aspx?csf=1&amp;web=1&amp;share=EXYMnkjq7sZCpRGeA8h0Mq4BlWmIRunbwj1ObSl_0JyOpw&amp;e=Q7vils&amp;cid=98098746-b9ac-4dc7-883f-b9df5d513349" target="_blank" class="first_a">営業</a></li>
<!--<a href="javascript:;" class="dekiru" onclick="javascript:window.open('http://www.dho.bc.obayashi.co.jp/new_seisan5up/','seisan5up','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,top=30,left=400');">できる！１０％UP</a>-->
<!--<li class="li_long_6"><a href="javascript:;" class="tdlist" onclick="javascript:window.open('http://mieruka-popup.fc.obayashi.co.jp/tza01/TZAS200/TZAS200','tdlist','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=750,height=210,top=30,left=400');">現場TODOリスト</a></li>-->
<!--<li class="li_long_5"><a href="javascript:;" class="kiduki" onclick="javascript:window.open('http://mieruka-popup.fc.obayashi.co.jp/tza01/TZAS100/TZAS100','kiduki','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=750,height=420,top=200,left=400');">気づき工程表</a></li>-->
<!--<li class="li_long"><a href="javascript:;" class="mieru" onclick="javascript:window.open('http://www.dho.bc.obayashi.co.jp/new_seisan5up/','seisan5up','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,top=30,left=400');">見える化システム</a>
</li>-->
    <li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.osm.obayashi.co.jp/portal/index.html" target="_blank" class="first_a">安全</a>
	</li><li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/TeamSite_techportal" target="_blank" class="first_a">技術</a>
		<a href="http://techforest.fc.obayashi.co.jp/tsc/" target="_blank" class="tech">TECHFOREST</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/design" target="_blank" class="first_a">設計</a></li>
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/dxportal" target="_blank" class="first_digital">ここから始める「デジタルおおばやし」</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://dms.fc.obayashi.co.jp/ock/" target="_blank" class="first_a">OC-ナレッジ</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.fc.obayashi.co.jp/convenience.html" target="_blank" class="first_a">便利帳</a></li>
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://gp.obayashi.co.jp/" target="_blank" class="first_a">大林グループ</a></li>
	</ul>
    <br class="clearfloat">
    

<div id="basicmenu" class="yuimenu yui-module yui-overlay yui-overlay-hidden" style="z-index: 1; position: absolute; visibility: hidden;">
<div class="bd">
        <ul class="first-of-type">
          <li id="yui-gen0" class="yuimenuitem first-of-type" groupindex="0" index="0"><a href="http://cbes.fc.obayashi.co.jp/cbes-dnavi/search?ui=custom_kbank_civil&amp;ctrl=refresh" target="_blank" class="know yuimenuitemlabel">土木</a></li>
          <li id="yui-gen1" class="yuimenuitem" groupindex="0" index="1"><a href="http://cbes.fc.obayashi.co.jp/cbes-dnavi/search?ui=custom_kbank_arch&amp;ctrl=refresh" target="_blank" class="know yuimenuitemlabel">建築</a></li>
          <li id="yui-gen2" class="yuimenuitem" groupindex="0" index="2"><a href="http://cbes.fc.obayashi.co.jp/cbes-dnavi/search?ui=custom_kbank_edb&amp;ctrl=refresh" target="_blank" class="know yuimenuitemlabel">環境</a></li>
        </ul>            
    </div>
</div>


  </div>
</div>

<div id="CollapsiblePanel2" class="CollapsiblePanel clearfix CollapsiblePanelOpen">
  <div class="CollapsiblePanelTab" tabindex="0">
  <h3><img src="images/side_h3_02.gif" alt="情報共有ツール" onclick="tabwrite2()"></h3>
  </div>
  <div class="CollapsiblePanelContent" style="display: block; visibility: visible; height: 100px;">
	<ul class="ul01">
	<li class="li_long_2"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/NaviPortal/SitePages/Office365.aspx" target="_blank" class="first_a">Ｏｆｆｉｃｅ３６５</a></li>
	<li class="li_normal"><a href="https://outlook.office365.com/owa/" target="_blank">　OWA</a></li>
	<li class="li_normal"><a href="https://obayashig.sharepoint.com/sites/NaviPortal/SitePages/Exchange_LargeFileSendRecieve.aspx" target="_blank">大容量ファイル</a></li> 
	<li class="li_normal"><a href="https://obayashig.sharepoint.com/sites/NaviPortal/SitePages/Exchange_GuestInvitation.aspx" target="_blank">　受付予約</a></li>
	<li class="li_normal" style="margin-left:0px;"><a href="https://obayashig.sharepoint.com/sites/NaviPortal/SitePages/IntraAndTool_SharePointSite.aspx" target="_blank">サイト</a></li> 
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://it.isc.obayashi.co.jp/tel_web/index.html" target="_blank" class="first_a">電話帳</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://lbs3.fc.obayashi.co.jp/msg/inf/web/asp/TWXA2Z.asp?GID=COM1000" target="_blank" class="first_a">各種設定</a></li> 
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.itsol.obayashi.co.jp/oc/comet/" target="_blank">OC-COMET</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://ocproject.fc.obayashi.co.jp/pj/site/main/login.aspx" target="_blank">OC-Project</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www2.group.obayashi.co.jp/it/box/index.html" target="_blank">Box</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www2.group.obayashi.co.jp/it/helpdesk/servicenow/ServiceNow_top.html" target="_blank">ServiceNow</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/sansan" target="_blank">Sansan</a></li>
	<li class="li_normal"></li>
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/TeamSite_0000_0505?e=1%3A592a48eb977645f8882bdae0d82ee7c5" target="_blank">Talent Palette</a></li>	
	</ul>
    <br class="clearfloat">
  </div>
</div>

<div id="CollapsiblePanel3" class="CollapsiblePanel clearfix CollapsiblePanelClosed">
  <div class="CollapsiblePanelTab" tabindex="0">
    <h3><img src="images/side_h3_03.gif" alt="業務処理" onclick="tabwrite3()"></h3>
  </div>
  <div class="CollapsiblePanelContent" style="display: none;">
	<ul class="ul01">
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="javascript:;" onclick="window.open('gyomusystemmenu.htm','','width=330,height=680')" class="first_a">業務システムメニュー</a></li>
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/dxdportal/systemnavi/" target="_blank" class="first_a">業務システムナビ</a></li>
    <li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/oelportal" target="_blank" class="first_a">eラーニング</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.pd.obayashi.co.jp/payroll/shuro/guide.htm" target="_blank" class="first_a">出勤簿</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.pd.obayashi.co.jp/jinji/jinjiportal/j_web.html" target="_blank" class="first_a">人事Web</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.itsol.obayashi.co.jp/keiri/keihi/keihi_top.htm" target="_blank" class="first_a">経費管理</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://b-plus.jtb-cwt.com/sso/SamlStart.aspx?CORPORATEID=a2h0TRNr%2fJo%24&amp;IDPNUMBER=s1FGbvm1qPk%24&amp;LANG=LFNXegKk4Hc%24&amp;SITE=iwIoZbLp1iQ%24&amp;pageHash=2CDF44852EF1AE9AC869E444359F521D" target="_blank" class="first_a">出張予約</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/dms/" target="_blank" class="first_a">文書管理</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.ga.obayashi.co.jp/ringi/ringi_toppage.htm" target="_blank" class="first_a">稟議</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://it.isc.obayashi.co.jp/ct_other/cost/hurikaejoho.html" target="_blank" class="first_a">振替情報</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="javascript:;" onclick="javascript:window.open('http://lbs3.fc.obayashi.co.jp/TZA/flashWin.asp','subwin','resizable,toolbar=no,menubar=no,width=320,height=650,top=30,left=50');" class="first_a">承認小窓</a></li>

	<p>&nbsp;</p>
	</ul>

    <div id="KMTmenu" class="yuimenuKMT yui-module yui-overlay yuimenu">
    <div class="bd">
        <ul class="first-of-type">
          <li id="yui-gen3" class="yuimenuitem first-of-type" groupindex="0" index="0"><a href="javascript:startdoboku()" class="know yuimenuitemlabel">土木技術</a></li>
          <li id="yui-gen4" class="yuimenuitem" groupindex="0" index="1"><a href="javascript:startkentiku()" class="know yuimenuitemlabel">建築技術</a></li>
        </ul>            
    </div>
    </div>
    
<script type="text/javascript">
<!--
function startdoboku(){
now = new Date();
hhh = now.getHours();
mmm = now.getMinutes();

if(mmm < 10){
	var mmm = "0" + mmm.toString();
}

var  time = (hhh.toString() + mmm.toString());

	if ( time < 200 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else if ( time > 2300 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else{
   window.open('http://kmt.fc.obayashi.co.jp/kmt/QAKmtSso.jsp?FunctionType=12&ModelType=10&ModelId=7');
	}
}

function startkentiku(){
now = new Date();
hhh = now.getHours();
mmm = now.getMinutes();

if(mmm < 10){
	var mmm = "0" + mmm.toString();
}

var  time = (hhh.toString() + mmm.toString());

	if ( time < 200 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else if ( time > 2300 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else{
   window.open('http://kmt.fc.obayashi.co.jp/kmt/QAKmtSso.jsp?FunctionType=12&ModelType=10&ModelId=1');
	}
}

/*
function startit(){
now = new Date();
hhh = now.getHours();
mmm = now.getMinutes();

if(mmm < 10){
	var mmm = "0" + mmm.toString();
}

var  time = (hhh.toString() + mmm.toString());

	if ( time < 200 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else if ( time > 2300 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else{
   window.open('http://kmt.fc.obayashi.co.jp/kmt/QAKmtSso.jsp?FunctionType=12&ModelType=10&ModelId=15');
	}
}
*/

-->
</script>
    
  </div>
</div>

<br class="clearfloat">

<!--BEGIN SOURCE CODE FOR EXAMPLE =============================== -->

<script type="text/javascript">

    /*
         Initialize and render the Menu when its elements are ready 
         to be scripted.
    */

    YAHOO.util.Event.onContentReady("basicmenu", function () {
    
        /*
             Instantiate a Menu:  The first argument passed to the 
             constructor is the id of the element in the page 
             representing the Menu; the second is an object literal 
             of configuration properties.
        */

        var oMenu = new YAHOO.widget.Menu("basicmenu");


        /*
             Call the "render" method with no arguments since the 
             markup for this Menu instance is already exists in the page.
        */

        oMenu.render();


        YAHOO.util.Event.addListener("menutoggle", "click", oMenu.show, null, oMenu);
    

    


var oMenu = new YAHOO.widget.Menu("KMTmenu");
YAHOO.util.Event.addListener("menutoggleKMT", "click", oMenu.show, null, oMenu);

    });
</script>
<!--END　技術相談室 =============================== -->
</div>







<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
<head>
  <meta http-equiv="content-style-type" content="text/css" />
  <title>電話帳・アドレス帳</title>
<style type="text/css">
 a { font-weight:normal; }
 a:link { color:#0000ff; }
 a:visited { color:#0000ff; }
 a:active { color:#ff0000; }
 td { text-align:left;font-weight:normal; }
 span.koumoku1 { color:#6090ef; }
 span.koumoku2 { color:#999999;font-size:small; }
 .ques { text-decoration:underline;color:#eeeeff; }
.auto-style2 {
	font-size: 16pt;
	color: #FF0000;
	font-weight: bold;
}
</style>
</head>
<body>
<div align="center">
<center>
<table border="2" cellpadding="4" cellspacing="0" style="border-collapse:collapse;width:100%;" bordercolor="#ffffff" bordercolordark="#cfe3ff" bordercolorlight="#cfe3ff">
  <tr>
    <td width="974" bordercolor="#ffffff" style="background-color:#f6f7fe;">
    <table cellpadding="0" cellspacing="0" style="width:100%;">
      <tr>
        <td width="65%">　　<font face="ＭＳ ゴシック" style="color:#595656;font-size:x-large;">電  話  帳・メールアドレス帳</font><font style="color:#ff0000;" size="4"><font face="ＭＳ ゴシック">［</font>社外秘<font face="ＭＳ ゴシック">]</font></font></td>
        <td width="35%"><a href="http://it.isc.obayashi.co.jp/pbx/" target="_blank" style="text-align:right;font-size:large;font-weight:bold;color:#db318f">内線電話ポータル</a></td>
<!--
        <td style="text-align:right;color:#db318f;font-size:large;font-weight:bold;">グループ会社も検索できるようになりました。</td>
-->
      </tr>
    </table>
    <!--<span style="color:#00f;margin-left:200px;">グループ会社電話帳は</span><a href="http://www.group.obayashi.co.jp/intro/info/tel/GP_tel_menu.html"><span style="color:#f00;text-decoration:underline;">こちら</span></a>-->
    </td>
  </tr>
  <tr>
    <td width="974" bordercolor="#ffffff" style="background-color:#edf5ff;text-align:center;">
      <font style="color:#01a040;" size="-1">氏名、メールアドレスは個人情報です。取扱いには充分ご注意ください。</font><font size="2">　　　　　　<a href="http://www.fc.obayashi.co.jp/" target="_top">トップメニューに戻る</a></font>&nbsp; 
    </td>
  </tr>
</table>
</center>
</div>

<hr color="#cfe3ff">



<div align="center">
<center>

<table border="3" cellspacing="0" style="width:900px;background-color:#cccccc;border-collapse:collapse;" bordercolor="#c0c0c0" bordercolordark="#cfe3ff" bordercolorlight="#cfe3ff">
  <tr style="background-color:#f0f0ff;">
    <td colspan="2" bordercolorlight="#ffffff" style="background-color:#4f86c5;">
      <font size="4" style="color:#ffffff;font-weight:bold;">&nbsp;&nbsp;電 話 帳</font></td>
  </tr>
  <tr>
    <td style="width:50%;background-color:#f2f2f2;">
      <table border="2" cellpadding="0" cellspacing="0" style="border-collapse:collapse;width:100%;" height="120" bordercolor="#111111" bordercolorlight="#ffffff" bordercolordark="#ffffff">
        <tr>
          <td>
            <font size="5">　</font><span class="koumoku1">■</span><a href="https://obayashig.phoneappli.net/sso" target="_blank" style="font-size:x-large;">大林グループ電話帳</a><br />
<!--
            <font size="5">　</font><span class="koumoku1">■</span><a href="http://09117dev.isc.obayashi.co.jp/msg/ref/kojin/Kojin_input.aspx" target="_blank" style="font-size:x-large;">氏名等を入力して検索</a><br />
-->
            <font size="2" style="color:#db318f;">　　　　組織ツリー、詳細検索</font></td>
        </tr>
        <tr>
          <td>
<!--
            <font size="5">　</font><span class="koumoku1">■</span><a href="http://09117dev.isc.obayashi.co.jp/msg/ref/web/asp/tel_index.aspx" target="_blank" style="font-size:x-large;">組織一覧から検索</a><br />
-->
            <span style="margin-left:10px;"><font size="4">旧電話帳</span><br />
            <span class="koumoku1" style="margin-left:20px;">■</span><a href="http://lbs3.fc.obayashi.co.jp/msg/ref/kojin/Kojin_input.aspx" target="_blank">氏名検索</a>
            <span class="koumoku1">■</span><a href="http://lbs3.fc.obayashi.co.jp/msg/ref/web/asp/tel_index.aspx" target="_blank">組織一覧検索</a>
            <span class="koumoku1">■</span><a href="http://lbs3.fc.obayashi.co.jp/msg/ref/kojin/KojinS_input.aspx" target="_blank">組織検索</a>
        </tr>
      </table>
    </td>
    <td style="width:50%;background-color:#f2f2f2;">
      <table border="2" cellpadding="0" cellspacing="0" style="border-collapse:collapse;width:100%;" height="120" bordercolor="#111111" bordercolordark="#ffffff" bordercolorlight="#ffffff">
        <tr>
           <td>
<!--
            　<span class="koumoku1">■</span><a href="http://09117dev.isc.obayashi.co.jp/msg/ref/kojin/KojinS_input.aspx" target="_blank">組織検索</a>
-->
            　<span class="koumoku1" style="color: #FF0000">■</span><a href="https://obayashig.sharepoint.com/sites/NaviPortal/SitePages/phoneapplipeople.aspx" target="_blank" class="auto-style2" style="color: #FF0000">大林グループ電話帳関連ページ(マニュアル)</a><br />
            <font size="6">　</font><br />

            　<span class="koumoku1">■</span><a href="https://obayashig.sharepoint.com/sites/NaviPortal/SitePages/tel_kyoyoshisetu.aspx" target="_blank">共用施設・執務フロア会議室　電話一覧</a><br />
          </td>
        </tr>
      </table>
    </td>
  </tr>
  <tr>
    <td colspan="2" style="background-color:#4f86c5;" bordercolorlight="#ffffff">
      <span style="color:#ffffff;"><font size="4" style="font-weight:bold;">&nbsp;&nbsp;緊急連絡先</font>（随時更新）</span>
    </td>
  </tr>
  <tr>
    <td colspan="2" style="background-color:#f2f2f2;height:50%;" bordercolorlight="#ffffff">
      <table border="0" cellpadding="0" cellspacing="0" style="border-collapse:collapse;width:100%;" bordercolor="#111111">
        <tr>
           <td style="width:50%;border-left:2px solid #ffffff;border-top:2px solid #ffffff;border-bottom:2px solid #ffffff;">
             <font size="5">　</font><span class="koumoku1">■</span><a href="http://www.ga.obayashi.co.jp/risk_mgt_top/kinkyu_renraku/risk_manage_renraku.htm" target="_blank"><span style="font-size:x-large;">緊急連絡網</span><font size="4">（店別）</font></a><br />
             <font size="2">　　　　<a href="http://www.fc.obayashi.co.jp/sub_menus/risk_manage.html" target="_parent">リスクマネジメント</a>－<a href="http://www.ga.obayashi.co.jp/risk_mgt_top/rs_committee.htm" target="_parent">危機管理体制</a> </font>
           </td>
           <td style="width:50%;border-right:2px solid #ffffff;border-top:2px solid #ffffff;border-bottom:2px solid #ffffff;">
            <font size="5">　</font><span class="koumoku1">■</span><a href="http://www.ga.obayashi.co.jp/shinsai/mainpage/shiryoshu/4_災害対策本部緊急連絡先一覧.pdf" target="_blank" style="font-weight:bold;"><span style="font-size:x-large;">災害対策本部緊急連絡網</a></font>
<br />
            <font size="2">　　　　<a href="http://www2.ga.obayashi.co.jp/e-portal/index.html"target="_parent">災害対策本部ホームページ</a></font>
          </td>
        </tr>
      </table>
    </td>
  </tr>
</table>

</center>
</div>
 <font size="1">　</font><br />
<div align="center">
<center>

<table border="3" cellspacing="1" bordercolor="#111111" bordercolorlight="#a3a2a4" bordercolordark="#a3a4a2" style="width:900px;border-collapse:collapse;">
  <tr>
    <td bordercolorlight="#ffffff" bordercolor="#808080" style="width:50%;background-color:#808080;color:#ffffff;font-weight:bold;">&nbsp;&nbsp;電話帳等の設定（組織・個人情報）</td>
    <td bordercolorlight="#ffffff" bordercolor="#808080" style="width:50%;background-color:#808080;color:#ffffff;font-weight:bold;">&nbsp;&nbsp;その他の機能</td>
  </tr>
  <tr>
    <td style="background-color:#eeece1;">
      <table border="2" cellpadding="0" cellspacing="0" style="border-collapse:collapse;width:100%;" bordercolor="#111111" bordercolorlight="#ffffff" bordercolordark="#ffffff" height="200">
        <tr>
          <td>
<!--
            　<span class="koumoku2">■</span><a href="http://09117dev.isc.obayashi.co.jp/msg/inf/web/asp/Redirect.asp?refer=PSN4000" target="_blank">あなたの電話番号の設定</a><br />
-->
            　<span class="koumoku2">■</span><a href="http://lbs3.fc.obayashi.co.jp/msg/inf/web/asp/Redirect.asp?refer=PSN4000" target="_blank">あなたの電話番号の設定</a><br />

            <font style="color:#4265ea;" size="-1">　　　あなたが所属する部署の電話番号を登録、変更できます。<br /><br /> </font>
<!--
            　<span class="koumoku2">■</span><a href="http://09117dev.isc.obayashi.co.jp/msg/app/web/asp/TWa2Z.asp?GID=ML1" target="_blank">あなたのメーリングリスト・共用メールボックスの設定</a><br />
-->
            　<span class="koumoku2">■</span><a href="http://www2.group.obayashi.co.jp/it/office365/outlook/16437/" target="_blank">あなたのメーリングリストの設定</a><br />

            <font style="color:#4265ea;" size="-1">　　　あなたが参加中・参加可能なメーリングリスト等を確認、設定できます。</font>
          </td>
        </tr>
        <tr>
          <td>
<!--
            　<span class="koumoku2">■</span><a href="http://09117dev.isc.obayashi.co.jp/msg/inf/web/asp/Redirect.asp?refer=PSN3000" target="_blank">個人情報の一括設定</a>（部門担当者用）　　<a href="http://09117dev.isc.obayashi.co.jp/msg/inf/web/help/infhelp.htm" target="_blank"><font size="2">利用方法</font></a><br />
-->
            　<span class="koumoku2">■</span><a href="http://lbs3.fc.obayashi.co.jp/msg/inf/web/asp/Redirect.asp?refer=PSN3000" target="_blank">個人情報の一括設定</a>（部門担当者用）　　<a href="http://soweb1v.isc.obayashi.co.jp/msg/inf/web/help/infhelp.htm" target="_blank"><font size="2">利用方法</font></a><br />

              <font style="color:#4265ea;" size="-1">　　　部門単位の電話番号や住所をまとめて設定<br /><br /> </font>
<!--
             　<span class="koumoku2">■</span><a href="http://09117dev.isc.obayashi.co.jp/msg/inf/web/asp/Redirect.asp?refer=ORG1000" target="_blank">部門電話情報等の設定</a>（部門担当者用）<br />
-->
             　<span class="koumoku2">■</span><a href="http://lbs3.fc.obayashi.co.jp/msg/inf/web/asp/Redirect.asp?refer=ORG1000" target="_blank">部門電話情報等の設定</a>（部門担当者用）<br />

              <font style="color:#4265ea;" size="-1">　　　組織の電話番号や住所の設定</font>
          </td>
        </tr>
      </table>
    </td>
    <td style="background-color:#eeece1;">
      <table border="2" cellpadding="0" cellspacing="0" style="border-collapse:collapse;width:100%;" height="200" bordercolor="#111111" bordercolorlight="#ffffff" bordercolordark="#ffffff">
        <tr>
          <td>
<!--
            　<span class="koumoku2">■</span><a href="http://09117dev.isc.obayashi.co.jp/msg/inf/web/asp/TWXA2Z.asp?GID=COM1000" target="_blank">メール等の各種設定　・　申請</a>　　　　<a href="http://09117dev.isc.obayashi.co.jp/msg/inf/web/help/infhelp.htm" target="_blank"><font size="2">利用方法</font></a><br />
-->
            　<span class="koumoku2">■</span><a href="http://lbs3.fc.obayashi.co.jp/msg/inf/web/asp/TWXA2Z.asp?GID=COM1000" target="_blank">メール等の各種設定　・　申請</a>　　　　<a href="http://lbs3.fc.obayashi.co.jp/msg/inf/web/help/infhelp.htm" target="_blank"><font size="2">利用方法</font></a><br />

             <font style="color:#4265ea;" size="-1">　　　パスワードの設定、協力スタッフの登録など</font><br />
             <font size="1">　</font><br />
<!--
             　<span class="koumoku2">■</span><a href="http://09117dev.isc.obayashi.co.jp/imeplus/ime_ks.asp?cn=&ou=">工事情報等のかな漢字変換辞書登録</a><br />
             　<span class="koumoku2">■</span><span style="color:#999">工事情報等のかな漢字変換辞書登録</span><br />
             <font style="color:#4265ea;" size="-1">　　　かな漢字変換辞書登録はサービスを終了しました。</font><br />
             <font size="1">　</font><br />
-->
             　<span class="koumoku2">■</span><a href="http://www2.group.obayashi.co.jp/it/office365/outlook/16437/" target="_blank">メーリングリスト</a><br />
             <font style="color:#4265ea;" size="-1">　　　メーリングリストの参加・投稿・開設・管理</font><br />
             <font size="1">　</font><br />
<!--
             　<span class="koumoku2">■</span><a href="http://09117dev.isc.obayashi.co.jp/msg/app/web/asp/TWa1Z.asp?GID=AD1" target="_blank">メールアドレス帳</a><br />
             　<span class="koumoku2">■</span><span style="color:#999">メールアドレス帳</span></a><br />

             <font style="color:#4265ea;" size="-1">　　　従来のメールアドレス帳はサービスを終了しました。<br />
　　　今後は<a href="http://www2.group.obayashi.co.jp/it/office365/outlook/13326/" target="_blank">階層型アドレス帳</a>をご利用ください。 </font><br />
             <font size="1">　</font><br />
-->
           　<span class="koumoku2">■</span><a href="address_search.html" target="_blank">メールアドレス検索システム</a>
          </td>
        </tr>
      </table>
    </td>
  </tr>
</table>


<table border="0" cellpadding="4" cellspacing="0" style="border-bottom:2px solid #999999;border-collapse:collapse;width:100%;" bordercolor="#111111">
  <tr>
    <td style="color:#333333;font-weight:bold;text-align:center;">&nbsp;&nbsp;問い合わせ先　<a href="http://www2.group.obayashi.co.jp/it/helpdesk/servicenow/ServiceNow_top.html"target="_blank">ICTサービスポータル</a></td>
  </tr>
</table>
</center>
</div>



<div align="center"><font size="-1">Copyright&copy; 2007-2020 OBAYASHI Corporation All Rights Reserved. </font></div>

</body>
</html>









<a href="address_search.html" target="_blank">メールアドレス検索システム</a>

// 2. 電話帳へのアクセス (直接URLを使用)
            $addressBookUrl = 'http://it.isc.obayashi.co.jp/tel_web/index.html';
            $addressBookResponse = $client->get($addressBookUrl);
            $addressBookHtml = (string) $addressBookResponse->getBody();
            $addressBookCrawler = new Crawler($addressBookHtml);

           // 3.「メールアドレス検索システム」リンクの抽出
            $mailAddressSearchLink = $addressBookCrawler->filter('a:contains("メールアドレス検索システム")')->attr('href');

            if (!$mailAddressSearchLink) {
                dd('メールアドレス検索システム link not found.');
            }

            // 絶対URLに変換
            $mailAddressSearchAbsoluteLink = $addressBookUrl . $mailAddressSearchLink; // 電話帳ページからの相対パスを考慮

            // 4. 「メールアドレス検索システム」ページへのアクセス
            $finalResponse = $client->get($mailAddressSearchAbsoluteLink);
            $finalHtml = (string) $finalResponse->getBody();

            dd('メールアドレス検索システム Page Content:', $finalHtml);






<div id="sidemenu"><a href="http://www.obayashi.co.jp/" target="_blank" id="to_obayashiHP"><img src="images/obayashiweb.gif" alt="社外ホームページへ" width="170" height="38"></a>
  <div id="h2_box">
<h2>サイドメニュー</h2>
<a href="javascript:;" onclick="blind_toggle.start(); return false;">設定</a>
<br class="clearfloat">
</div>

<div id="slide_setmenu" style="overflow: hidden; visibility: visible; display: none;">
<div class="sp01">
サイドメニューの展開状態を<br>
保存しますか？
</div>
<form name="form">
<div class="sp02">
  <input type="radio" name="set_sidemenu" value="keep" id="set_sidemenu_0"><span>保持する</span>
  <br>
  <input type="radio" name="set_sidemenu" value="close" id="set_sidemenu_1"><span>起動時はいつも閉じる</span>
  <br>
  <input type="radio" name="set_sidemenu" value="open" id="set_sidemenu_2"><span>起動時はいつも開く</span>
</div>
<div class="sp03">
	<input name="set" type="button" value="設定する" onclick="CookieCheck(),blind_toggle.start(); return false;">
	<input name="set" type="button" value="中止" onclick="blind_toggle.start(); return false;">
</div>
</form>
</div>

<ul id="list01" class="ul01">
<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.csr.obayashi.co.jp/topics/assets_images/kihon_rinen_2018rev.pdf" target="_blank" class="first_a">基本理念</a></li>

<li class="li_normal_sec"><img src="images/bulet_side.gif" class="bulet"><a href="http://secure.fc.obayashi.co.jp/accdept/sougoukikaku/log/clicklog.asp?url=http://secure.fc.obayashi.co.jp/accdept/sougoukikaku/overdirectorandauditor/index.htm&amp;title=keiei" target="_blank" class="first_a">経営情報</a></li>


<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.brand.obayashi.co.jp/pdf/DL_slogan_A4.pdf" target="_blank" class="first_a">ブランドビジョン</a></li>

<li class="li_long"><strong><font size="1"><img src="images/bulet_side.gif" class="bulet"><a href="http://gdi.obayashi.co.jp/chukei2022/" target="_blank" class="first_a">中期経営計画２０２２ポータル</a></font></strong></li>
<li class="li_long_5"><a href="http://secure.fc.obayashi.co.jp/accdept/sougoukikaku/newhomepage/03%20keieikeikaku/2025sisaku.pdf" class="rinri_4" target="_blank">２０２５年度経営施策</a></li>

<li class="li_long_2"><img src="images/bulet_side.gif" class="bulet"><a href="http://gdi.obayashi.co.jp/seisansei/mtg.html" target="_blank" class="first_c">会議改革ポータル</a></li>

<br>

<li class="li_long_2"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.ga.obayashi.co.jp/tekiseika/kigyourinri/kigyourinri.htm" target="_blank" class="first_c">企業倫理</a></li>
<li class="li_long_5"><a href="http://www.ga.obayashi.co.jp/rinritsuho/tsuho.pdf" class="rinri_4" target="_blank">企業倫理相談・通報制度</a></li>

<li class="li_long_5"><a href="http://www.ga.obayashi.co.jp/tekiseika/kigyourinri/kaigoukonsinrepo.html" class="rinri_4" target="_blank">同業者会合・懇親会報告</a></li>

<li class="li_long_5"><a href="http://www.ga.obayashi.co.jp/tekiseika/kigyourinri/training/portal/top.html" class="rinri_4" target="_blank">職場内研修ポータル</a></li>

<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://gdi.obayashi.co.jp/" target="_blank" class="first_digital">ダイバーシティ＆インクルージョン</a></li>

<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.ga.obayashi.co.jp/kikikanri/kikikanri.htm" target="_blank" class="first_a">危機管理</a></li>
	<li class="li_normal" style="margin-right:-3px;"><img src="images/bulet_side.gif" class="bulet"><span style="font-size:95%"><a href="http://www2.ga.obayashi.co.jp/e-portal/index.html" target="_blank" class="second_a">災害対策本部</a></span></li>
<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/harassment" target="_blank" class="first_a">ハラスメント</a></li>
<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><span style="font-size:95%"><a href="http://www.ga.obayashi.co.jp/rinritsuho/soudan.html" target="_blank" class="first_a">各種相談窓口</a></span></li>
</ul>
<br class="clearfloat">

<div id="CollapsiblePanel1" class="CollapsiblePanel clearfix CollapsiblePanelClosed">

  <div class="CollapsiblePanelTab" tabindex="1">
  <h3><img src="images/side_h3_01.gif" alt="ポータル" onclick="tabwrite1()"></h3>
  </div>
  <div class="CollapsiblePanelContent2" style="display: none;">
	<ul class="ul01">
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.pd.obayashi.co.jp/jinji/jinjiportal/" target="_blank" class="first_a">人事給与</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.pd.obayashi.co.jp/jinji/work_innovation/" target="_blank" class="first_a">働き方改革</a></li>
		<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.pd.obayashi.co.jp/jinji/recruit/" target="_blank" class="first_a">学生の社員訪問申請</a></li>
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.dho.cvl.obayashi.co.jp/c-portal/" target="_blank" class="first_a">土木</a><a href="javascript:;" class="ichiniti" onclick="javascript:window.open('http://www.dho.cvl.obayashi.co.jp/new_ichinichi_ichiwa/','cvlskk','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,top=30,left=400');"><img src="images/civil.gif" alt="土木一日一話" width="65" height="11"></a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.dho2.bc.obayashi.co.jp/" target="_blank" class="first_a">建築</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/eigyo_portal/SitePages/Home.aspx?csf=1&amp;web=1&amp;share=EXYMnkjq7sZCpRGeA8h0Mq4BlWmIRunbwj1ObSl_0JyOpw&amp;e=Q7vils&amp;cid=98098746-b9ac-4dc7-883f-b9df5d513349" target="_blank" class="first_a">営業</a></li>
<!--<a href="javascript:;" class="dekiru" onclick="javascript:window.open('http://www.dho.bc.obayashi.co.jp/new_seisan5up/','seisan5up','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,top=30,left=400');">できる！１０％UP</a>-->
<!--<li class="li_long_6"><a href="javascript:;" class="tdlist" onclick="javascript:window.open('http://mieruka-popup.fc.obayashi.co.jp/tza01/TZAS200/TZAS200','tdlist','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=750,height=210,top=30,left=400');">現場TODOリスト</a></li>-->
<!--<li class="li_long_5"><a href="javascript:;" class="kiduki" onclick="javascript:window.open('http://mieruka-popup.fc.obayashi.co.jp/tza01/TZAS100/TZAS100','kiduki','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=750,height=420,top=200,left=400');">気づき工程表</a></li>-->
<!--<li class="li_long"><a href="javascript:;" class="mieru" onclick="javascript:window.open('http://www.dho.bc.obayashi.co.jp/new_seisan5up/','seisan5up','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,top=30,left=400');">見える化システム</a>
</li>-->
    <li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.osm.obayashi.co.jp/portal/index.html" target="_blank" class="first_a">安全</a>
	</li><li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/TeamSite_techportal" target="_blank" class="first_a">技術</a>
		<a href="http://techforest.fc.obayashi.co.jp/tsc/" target="_blank" class="tech">TECHFOREST</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/design" target="_blank" class="first_a">設計</a></li>
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/dxportal" target="_blank" class="first_digital">ここから始める「デジタルおおばやし」</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://dms.fc.obayashi.co.jp/ock/" target="_blank" class="first_a">OC-ナレッジ</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.fc.obayashi.co.jp/convenience.html" target="_blank" class="first_a">便利帳</a></li>
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="http://gp.obayashi.co.jp/" target="_blank" class="first_a">大林グループ</a></li>
	</ul>
    <br class="clearfloat">
    

<div id="basicmenu" class="yuimenu yui-module yui-overlay yui-overlay-hidden" style="z-index: 1; position: absolute; visibility: hidden;">
<div class="bd">
        <ul class="first-of-type">
          <li id="yui-gen0" class="yuimenuitem first-of-type" groupindex="0" index="0"><a href="http://cbes.fc.obayashi.co.jp/cbes-dnavi/search?ui=custom_kbank_civil&amp;ctrl=refresh" target="_blank" class="know yuimenuitemlabel">土木</a></li>
          <li id="yui-gen1" class="yuimenuitem" groupindex="0" index="1"><a href="http://cbes.fc.obayashi.co.jp/cbes-dnavi/search?ui=custom_kbank_arch&amp;ctrl=refresh" target="_blank" class="know yuimenuitemlabel">建築</a></li>
          <li id="yui-gen2" class="yuimenuitem" groupindex="0" index="2"><a href="http://cbes.fc.obayashi.co.jp/cbes-dnavi/search?ui=custom_kbank_edb&amp;ctrl=refresh" target="_blank" class="know yuimenuitemlabel">環境</a></li>
        </ul>            
    </div>
</div>


  </div>
</div>

<div id="CollapsiblePanel2" class="CollapsiblePanel clearfix CollapsiblePanelOpen">
  <div class="CollapsiblePanelTab" tabindex="0">
  <h3><img src="images/side_h3_02.gif" alt="情報共有ツール" onclick="tabwrite2()"></h3>
  </div>
  <div class="CollapsiblePanelContent" style="display: block; visibility: visible; height: 100px;">
	<ul class="ul01">
	<li class="li_long_2"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/NaviPortal/SitePages/Office365.aspx" target="_blank" class="first_a">Ｏｆｆｉｃｅ３６５</a></li>
	<li class="li_normal"><a href="https://outlook.office365.com/owa/" target="_blank">　OWA</a></li>
	<li class="li_normal"><a href="https://obayashig.sharepoint.com/sites/NaviPortal/SitePages/Exchange_LargeFileSendRecieve.aspx" target="_blank">大容量ファイル</a></li> 
	<li class="li_normal"><a href="https://obayashig.sharepoint.com/sites/NaviPortal/SitePages/Exchange_GuestInvitation.aspx" target="_blank">　受付予約</a></li>
	<li class="li_normal" style="margin-left:0px;"><a href="https://obayashig.sharepoint.com/sites/NaviPortal/SitePages/IntraAndTool_SharePointSite.aspx" target="_blank">サイト</a></li> 
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://it.isc.obayashi.co.jp/tel_web/index.html" target="_blank" class="first_a">電話帳</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://lbs3.fc.obayashi.co.jp/msg/inf/web/asp/TWXA2Z.asp?GID=COM1000" target="_blank" class="first_a">各種設定</a></li> 
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.itsol.obayashi.co.jp/oc/comet/" target="_blank">OC-COMET</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://ocproject.fc.obayashi.co.jp/pj/site/main/login.aspx" target="_blank">OC-Project</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www2.group.obayashi.co.jp/it/box/index.html" target="_blank">Box</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www2.group.obayashi.co.jp/it/helpdesk/servicenow/ServiceNow_top.html" target="_blank">ServiceNow</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/sansan" target="_blank">Sansan</a></li>
	<li class="li_normal"></li>
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/TeamSite_0000_0505?e=1%3A592a48eb977645f8882bdae0d82ee7c5" target="_blank">Talent Palette</a></li>	
	</ul>
    <br class="clearfloat">
  </div>
</div>

<div id="CollapsiblePanel3" class="CollapsiblePanel clearfix CollapsiblePanelClosed">
  <div class="CollapsiblePanelTab" tabindex="0">
    <h3><img src="images/side_h3_03.gif" alt="業務処理" onclick="tabwrite3()"></h3>
  </div>
  <div class="CollapsiblePanelContent" style="display: none;">
	<ul class="ul01">
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="javascript:;" onclick="window.open('gyomusystemmenu.htm','','width=330,height=680')" class="first_a">業務システムメニュー</a></li>
	<li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/dxdportal/systemnavi/" target="_blank" class="first_a">業務システムナビ</a></li>
    <li class="li_long"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/oelportal" target="_blank" class="first_a">eラーニング</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.pd.obayashi.co.jp/payroll/shuro/guide.htm" target="_blank" class="first_a">出勤簿</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.pd.obayashi.co.jp/jinji/jinjiportal/j_web.html" target="_blank" class="first_a">人事Web</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.itsol.obayashi.co.jp/keiri/keihi/keihi_top.htm" target="_blank" class="first_a">経費管理</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://b-plus.jtb-cwt.com/sso/SamlStart.aspx?CORPORATEID=a2h0TRNr%2fJo%24&amp;IDPNUMBER=s1FGbvm1qPk%24&amp;LANG=LFNXegKk4Hc%24&amp;SITE=iwIoZbLp1iQ%24&amp;pageHash=2CDF44852EF1AE9AC869E444359F521D" target="_blank" class="first_a">出張予約</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="https://obayashig.sharepoint.com/sites/dms/" target="_blank" class="first_a">文書管理</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://www.ga.obayashi.co.jp/ringi/ringi_toppage.htm" target="_blank" class="first_a">稟議</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="http://it.isc.obayashi.co.jp/ct_other/cost/hurikaejoho.html" target="_blank" class="first_a">振替情報</a></li>
	<li class="li_normal"><img src="images/bulet_side.gif" class="bulet"><a href="javascript:;" onclick="javascript:window.open('http://lbs3.fc.obayashi.co.jp/TZA/flashWin.asp','subwin','resizable,toolbar=no,menubar=no,width=320,height=650,top=30,left=50');" class="first_a">承認小窓</a></li>

	<p>&nbsp;</p>
	</ul>

    <div id="KMTmenu" class="yuimenuKMT yui-module yui-overlay yuimenu">
    <div class="bd">
        <ul class="first-of-type">
          <li id="yui-gen3" class="yuimenuitem first-of-type" groupindex="0" index="0"><a href="javascript:startdoboku()" class="know yuimenuitemlabel">土木技術</a></li>
          <li id="yui-gen4" class="yuimenuitem" groupindex="0" index="1"><a href="javascript:startkentiku()" class="know yuimenuitemlabel">建築技術</a></li>
        </ul>            
    </div>
    </div>
    
<script type="text/javascript">
<!--
function startdoboku(){
now = new Date();
hhh = now.getHours();
mmm = now.getMinutes();

if(mmm < 10){
	var mmm = "0" + mmm.toString();
}

var  time = (hhh.toString() + mmm.toString());

	if ( time < 200 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else if ( time > 2300 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else{
   window.open('http://kmt.fc.obayashi.co.jp/kmt/QAKmtSso.jsp?FunctionType=12&ModelType=10&ModelId=7');
	}
}

function startkentiku(){
now = new Date();
hhh = now.getHours();
mmm = now.getMinutes();

if(mmm < 10){
	var mmm = "0" + mmm.toString();
}

var  time = (hhh.toString() + mmm.toString());

	if ( time < 200 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else if ( time > 2300 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else{
   window.open('http://kmt.fc.obayashi.co.jp/kmt/QAKmtSso.jsp?FunctionType=12&ModelType=10&ModelId=1');
	}
}

/*
function startit(){
now = new Date();
hhh = now.getHours();
mmm = now.getMinutes();

if(mmm < 10){
	var mmm = "0" + mmm.toString();
}

var  time = (hhh.toString() + mmm.toString());

	if ( time < 200 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else if ( time > 2300 ){
   alert("利用時間外です。（利用時間 02:00～23:00）..");
	}else{
   window.open('http://kmt.fc.obayashi.co.jp/kmt/QAKmtSso.jsp?FunctionType=12&ModelType=10&ModelId=15');
	}
}
*/

-->
</script>
    
  </div>
</div>

<br class="clearfloat">

<!--BEGIN SOURCE CODE FOR EXAMPLE =============================== -->

<script type="text/javascript">

    /*
         Initialize and render the Menu when its elements are ready 
         to be scripted.
    */

    YAHOO.util.Event.onContentReady("basicmenu", function () {
    
        /*
             Instantiate a Menu:  The first argument passed to the 
             constructor is the id of the element in the page 
             representing the Menu; the second is an object literal 
             of configuration properties.
        */

        var oMenu = new YAHOO.widget.Menu("basicmenu");


        /*
             Call the "render" method with no arguments since the 
             markup for this Menu instance is already exists in the page.
        */

        oMenu.render();


        YAHOO.util.Event.addListener("menutoggle", "click", oMenu.show, null, oMenu);
    

    


var oMenu = new YAHOO.widget.Menu("KMTmenu");
YAHOO.util.Event.addListener("menutoggleKMT", "click", oMenu.show, null, oMenu);

    });
</script>
<!--END　技術相談室 =============================== -->


</div>


<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Frameset//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-frameset.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">

<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="ROBOTS" content="NOINDEX, FOLLOW" />
    <title>大林組イントラネット</title>
    <SCRIPT language="JavaScript" SRC="http://www.fc.obayashi.co.jp/accessCookie.js"></SCRIPT>

    <script language="JavaScript1.2">
        <!--
        window.onload=init
        
        ie=false
        nn=false
        if(document.all){ie=true}
        if(navigator.appName=="Netscape"||navigator.userAgent.indexOf("Opera")!=-1){nn=true}
        
        function init(){
        if(ie){
        frames[0].document.body.onscroll=scrollie
        frames[1].document.body.onscroll=scrollie
        }
        if(nn){
        scroll=new Array(0,0)
        scrollnn()
        }
        }
        
        function scrollie(){
        if(frames[0].event){
        frames[1].scrollTo(0,frames[0].document.body.scrollTop)
        }
        if(frames[1].event){
        frames[0].scrollTo(0,frames[1].document.body.scrollTop)
        }
        }
        
        function scrollnn(){
        var scr0=frames[0].pageYOffset
        var scr1=frames[1].pageYOffset
        if(scr0!=scroll[0]){
        //左がスクロール
        frames[1].scrollTo(0,scr0)
        scroll[0]=scr0
        scroll[1]=scr0
        }else{
        if(scr1!=scroll[1]){
        //右がスクロール
        frames[0].scrollTo(0,scr1)
        scroll[0]=scr1
        scroll[1]=scr1
        }}
        setTimeout("scrollnn()",0)
        }
        //-->
    </script>
</head>

<frameset cols="172,*" frameborder="no" border="0" framespacing="0">
    <frame src="/top/sidemenu.html" name="leftFrame" id="leftFrame" title="サイドメニュー" noresize="noresize"
        scrolling="no" />
    <frame src="/top/main.html" name="mainFrame" id="mainFrame" title="mainFrame" noresize="noresize" />
</frameset>
<noframes>

    <body>
    </body>
</noframes>

</html>







<HTML>

<HEAD>
    <TITLE>データベースへの問い合わせ中</TITLE>
    <META HTTP-EQUIV="Content-Type" Content="text/html; charset=x-sjis">

    <script type="text/javascript" src="./jquery-1.10.2.min.js"></script>

    <script language="JavaScript">

        //2021/12/08 S
        //建築ポータル起動有無設定読み込み
        var strDhoStatus = getCookie('dhostatus');
        //2021/12/08 E

        //window.open("waitingSub.htm","subwin","resizable,toolbar=no,menubar=no,width=300,height=170,top=50,left=50");
        sai = getCookie('stopsystem').charAt(0);
        //承認小窓はCookie"stopsystem"の1桁目が"1"のときは表示しない
        if (sai != '1') {
            //window.open("waitingSub.htm","subwin","resizable,toolbar=no,menubar=no,width=320,height=550,top=50,left=50");
            //2022/12/28 S
            //window.open("waitingSub.htm","subwin","resizable=no,toolbar=no,menubar=no,width=320,height=650,top=30,left=20");
            window.open("waitingSub.htm", "subwin", "resizable=no,toolbar=no,menubar=no,width=320,height=420,top=30,left=20");
            //2022/12/28 E
        }
        //土木一日一話 → 土木ポータル(2024/04/03)
        sec = parseInt(getCookie('secCode'));
        shokushu = right(getCookie('multiCode'), 3);
        //2023/03/17 S 特定の部署に本務所属するユーザも起動可能となるよう改修
        var strSyoCode = getCookie('syoCode');

        //if(sec <= 11 && (shokushu == '110' || shokushu == '111' || shokushu == '140' || shokushu == '141')){
        if ((sec <= 11 && (shokushu == '110' || shokushu == '111' || shokushu == '140' || shokushu == '141')) || (strSyoCode == '09231' || strSyoCode == '09232' || strSyoCode == '09243' || strSyoCode == '09244' || strSyoCode == '09246' || strSyoCode == '09247')) {
            //2023/03/17 E

            //2019/06/12 起動URL、起動設定変更S
            //window.open("http://www.dho.cvl.obayashi.co.jp/civil_portal01/ichinichi_ichiwa/skk_ad.html","cvlskk", "width=500,height=570,top=30,left=400,scrollbars=no,resizable=no,menubar=no,toolbar=no");
            //window.open('http://www.dho.cvl.obayashi.co.jp/new_ichinichi_ichiwa/','cvlskk','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,top=30,left=400');
            //2019/06/12 起動URL、起動設定変更 E

            //2024/04/03 土木ポータル変更S
            //左位置
            var subx = 400;
            //上位置
            var suby = 30;
            //幅
            var subw = 1300;
            //高さ
            var subh = 800;

            //画面幅取得
            var intWinW = window.screen.width;

            if (intWinW < 1900) {
                //画面サイズが1900未満の場合
                subw = 910;
                subh = 600;
            }

            window.open("http://www.dho.cvl.obayashi.co.jp/c-portal/", "cvlskk", "width=" + subw + ",height=" + subh + ",top=" + suby + ",left=" + subx + ",scrollbars=yes,resizable=no,menubar=no,toolbar=no");

            //2024/04/03 土木ポータル変更E

        }
        //生産性５％アップ
        ccc = '';
        sec = parseInt(getCookie('secCode'));
        jyu = parseInt(getCookie('jyuCode'), 10);
        shokushu = right(getCookie('multiCode'), 3);
        uid = getCookie('userID');
        var jg = getCookie('multiCode').substring(0, 1);
        //2021/10/18 S 起動廃止
        //if(uid != '68199' && sec <= 11 && (jyu >= 5 && jyu <= 74) && ((shokushu == '100' && ccc == '3')||(shokushu == '101' && ccc == '3')|| shokushu == '130' || shokushu == '131' ||(shokushu == '140' && (ccc == '3' || jg == '0')) ||(shokushu == '141' && ccc == '3')|| shokushu == '151' || shokushu == '152' || shokushu == '153' || shokushu == '154')){
        //    window.open('http://www.dho.bc.obayashi.co.jp/new_seisan5up/','seisan5up','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,top=30,left=360');
        //}
        //2021/10/18 E

        //2021/10/18 S 新建築ポータル起動処理実装
        var strOpt = 'top=30,left=360,scrollbars=yes,resizable=yes,menubar=yes,location=yes,directories=yes,status=yes';
        //2023/02/22 S 建築ポータル起動条件変更対応
        //if(uid != '68199' && sec <= 11 && (jyu >= 5 && jyu <= 74) && ((shokushu == '100' && ccc == '3')||(shokushu == '101' && ccc == '3')|| shokushu == '130' || shokushu == '131' ||(shokushu == '140' && (ccc == '3' || jg == '0')) ||(shokushu == '141' && ccc == '3')|| shokushu == '151' || shokushu == '152' || shokushu == '153' || shokushu == '154')){
        if (uid != '68199' && sec <= 15 && ((jyu >= 5 && jyu <= 74) || jyu == 90 || jyu == 94 || jyu == 95) && ((shokushu == '100' && ccc == '3') || (shokushu == '101' && ccc == '3') || shokushu == '130' || shokushu == '131' || (shokushu == '140' && (ccc == '3' || jg == '0')) || (shokushu == '141' && ccc == '3') || shokushu == '151' || shokushu == '152' || shokushu == '153' || shokushu == '154')) {
            //2023/02/22 E
            //旧「できる！10%UP(さらに昔の生産性５％アップ)」起動条件と合致する場合は新建築ポータルを起動

            //2021/12/08 S
            //window.open('http://www.dho2.bc.obayashi.co.jp/','dho2',strOpt);
            //cookie値(dhostatus)が'1'でない場合、建築ポータルを起動。'1'の場合は起動しない
            if (strDhoStatus != '1') { window.open('http://www.dho2.bc.obayashi.co.jp/', 'dho2', strOpt); }
            //2021/12/08 E

        } else {
            var blnChkTza01 = CheckTza01();
            if (blnChkTza01) {
                //建築ポップアップの起動条件と合致する場合は新建築ポータルを起動

                //2021/12/08 S
                //window.open('http://www.dho2.bc.obayashi.co.jp/','dho2',strOpt);
                //cookie値(dhostatus)が'1'でない場合、建築ポータルを起動。'1'の場合は起動しない
                if (strDhoStatus != '1') { window.open('http://www.dho2.bc.obayashi.co.jp/', 'dho2', strOpt); }
                //2021/12/08 E
            }
        }
        //2021/10/18 E

        //RNイントラPOPUP S
        //2020/06/05 バイパスするよう設定
        //    if (HttpRequest_Send("http://toraburun.fc.obayashi.co.jp/RNPAPI/API/RNPA001.aspx", null) == '1') {
        if (HttpRequest_Send("ReqTrans.aspx?p=http://toraburun.fc.obayashi.co.jp/RNPAPI/API/RNPA001.aspx", null) == '1') {
            //2020/06/05
            //画面高さ取得
            var intWinH = window.screen.height;
            //900を超えている場合は、高さ900とする。900未満の場合はタスクバー領域等を考慮し、少し縮めた高さに設定
            if (intWinH > 900) { intWinH = 900; } else { intWinH += -50; }
            //起動
            window.open("http://toraburun.fc.obayashi.co.jp/RNP/frm/RNPS001.aspx", "rnp", "width=576,height=" + intWinH + ",top=30,left=780,scrollbars=yes,resizable=no,menubar=no,toolbar=no");
        }
        //RNイントラPOPUP E

        //2023/12/05 トラブルＤＢ S
        if (HttpRequest_Send("ReqTrans.aspx?p=http://toraburun.fc.obayashi.co.jp/toradbapi/TDBPA001.aspx", null) == '1') {
            var subw = 632;
            var subh = 446;
            var subx = (window.screen.width - subw) / 2;
            var sbuy = (window.screen.height - subh) / 2;

            window.open("http://www.brc.osk.obayashi.co.jp/troubleDB/trial/troubldb_popup.html", "tdbp", "width=" + subw + ",height=" + subh + ",top=" + sbuy + ",left=" + subx + ",scrollbars=yes,resizable=no,menubar=no,toolbar=no");

        }
        //2023/12/05 トラブルＤＢ E

        //2021/10/18 S 起動廃止
        //Exec_Tza01();
        //2021/10/18 S

        window.location.href = "http://www.fc.obayashi.co.jp";

        function getCookie(key) {
            tmp = document.cookie + ";";
            tmp1 = tmp.indexOf(key, 0);
            if (tmp1 != -1) {
                tmp = tmp.substring(tmp1, tmp.length);
                start = tmp.indexOf("=", 0) + 1;
                end = tmp.indexOf(";", start);
                return (unescape(tmp.substring(start, end)));
            }
            return ("");
        }
        //
        // 文字列を右側から指定文字数だけ取り出す関数
        //
        function right(str, n) {
            l = str.length;
            if (n > l) n = l;
            return (str.substring(l - n, l));
        }
        //RNイントラPOPUP S
        //通信処理
        function HttpRequest_Send(strUrl, strParam) {
            var ajax = null;

            for (i = 0; i < 3; i++) {
                try {
                    if (window.XMLHttpRequest) { ajax = new XMLHttpRequest(); }
                    else if (window.ActiveXObject) {
                        try {
                            ajax = new ActiveXObject("Msxml2.XMLHTTP");
                        } catch (e) {
                            ajax = new ActiveXObject("Microsoft.XMLHTTP");
                        }
                    }
                    //通信実行
                    ajax.open("POST", strUrl, false);
                    ajax.timeout = 5000;
                    ajax.send(strParam);
                    return ajax.responseText;

                } catch (e) {
                    if (i == 2) {
                        return '0';
                    } else {
                        //alert(i + e.message);
                        continue;
                    }
                } finally {
                    ajax = null;
                }
            }
        }
        //RNイントラPOPUP E

        //2021/10/18 S 改修
        //TZA01アクセス権限認証
        function CheckTza01() {
            //参照先サーバ(★導入環境に合わせて設定変更)
            var strRootUrl = 'http://mieruka-popup.fc.obayashi.co.jp';

            //WebAPIに対するURL設定
            var strUrl = 'ReqTrans.aspx?p=' + strRootUrl + '/tza01/TZAA000/TZAA000';

            try {
                //通信実行
                var objRet = sendQuery(strUrl, '');

                //結果確認(エラー、もしくはアクセス権なしの場合は処理抜ける)
                if (objRet == undefined || objRet == null) { return false; }

                var objRetJson = (new Function("return " + objRet))();

                if (objRetJson.ErrCode != 0 || objRetJson.RetCode != 1) { return false; }

                //認証ＯＫの場合はTrueを返却
                return true;

            } catch (e) {
                //処理を止めない為、スルー
                return false;
            }
        }
        //2021/10/18 E

        //通信(同期)
        function sendQuery(send_url, send_data) {
            var ret_data;

            $.ajax({
                type: 'POST',
                async: false,
                url: send_url,
                data: send_data,
                success: function (data) {
                    ret_data = data;
                }
            });
            return ret_data
        }
        //2020/05/28 E

    </SCRIPT>
</HEAD>

<BODY>
    <CENTER>
        <H3>処理中です。しばらくお待ち下さい。</H3>
    </CENTER>
</BODY>

</HTML>




            
^ "Login failed."
^ """

<HTML>
<HEAD>
<TITLE>データベースへの問い合わせ中</TITLE>
<META HTTP-EQUIV="Content-Type" Content="text/html; charset=x-sjis">

<script type="text/javascript" src="./jquery-1.10.2.min.js"></script>

<script language="JavaScript">

    //2021/12/08 S
    //建築ポータル起動有無設定読み込み
    var strDhoStatus = getCookie('dhostatus');
    //2021/12/08 E
    
\t//window.open("waitingSub.htm","subwin","resizable,toolbar=no,menubar=no,width=300,height=170,top=50,left=50");
    sai = getCookie('stopsystem').charAt(0);
    //承認小窓はCookie"stopsystem"の1桁目が"1"のときは表示しない
   \tif(sai != '1'){
   \t    //window.open("waitingSub.htm","subwin","resizable,toolbar=no,menubar=no,width=320,height=550,top=50,left=50");
        //2022/12/28 S
   \t    //window.open("waitingSub.htm","subwin","resizable=no,toolbar=no,menubar=no,width=320,height=650,top=30,left=20");
   \t    window.open("waitingSub.htm","subwin","resizable=no,toolbar=no,menubar=no,width=320,height=420,top=30,left=20");
        //2022/12/28 E
\t}
\t//土木一日一話 → 土木ポータル(2024/04/03)
\tsec = parseInt(getCookie('secCode'));
\tshokushu = right(getCookie('multiCode'),3);
\t//2023/03/17 S 特定の部署に本務所属するユーザも起動可能となるよう改修
\tvar strSyoCode = getCookie('syoCode');

\t//if(sec <= 11 && (shokushu == '110' || shokushu == '111' || shokushu == '140' || shokushu == '141')){
\tif((sec <= 11 && (shokushu == '110' || shokushu == '111' || shokushu == '140' || shokushu == '141')) || (strSyoCode == '09231' || strSyoCode == '09232' || str ▶
\t    //2023/03/17 E

        //2019/06/12 起動URL、起動設定変更S
\t    //window.open("http://www.dho.cvl.obayashi.co.jp/civil_portal01/ichinichi_ichiwa/skk_ad.html","cvlskk", "width=500,height=570,top=30,left=400,scrollbars=n ▶
\t    //window.open('http://www.dho.cvl.obayashi.co.jp/new_ichinichi_ichiwa/','cvlskk','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,t ▶
\t    //2019/06/12 起動URL、起動設定変更 E

      //2024/04/03 土木ポータル変更S
        //左位置
        var subx = 400;
        //上位置
        var suby = 30;        
        //幅
        var subw = 1300;
        //高さ
        var subh = 800;
                    
        //画面幅取得
        var intWinW = window.screen.width;

        if (intWinW < 1900){
            //画面サイズが1900未満の場合
            subw = 910;
            subh = 600;
        }

        window.open("http://www.dho.cvl.obayashi.co.jp/c-portal/", "cvlskk", "width=" + subw + ",height=" + subh + ",top=" + suby + ",left=" + subx + ",scrollba ▶

      //2024/04/03 土木ポータル変更E

\t}
\t//生産性５％アップ
\tccc = '';
\tsec = parseInt(getCookie('secCode'));
\tjyu = parseInt(getCookie('jyuCode'),10);
\tshokushu = right(getCookie('multiCode'),3);
\tuid = getCookie('userID');
    var jg = getCookie('multiCode').substring(0,1);
    //2021/10/18 S 起動廃止
    //if(uid != '68199' && sec <= 11 && (jyu >= 5 && jyu <= 74) && ((shokushu == '100' && ccc == '3')||(shokushu == '101' && ccc == '3')|| shokushu == '130' ||  ▶
    //    window.open('http://www.dho.bc.obayashi.co.jp/new_seisan5up/','seisan5up','resizable=no,scrollbars=yes,menubar=no,toolbar=no,width=576,height=768,top= ▶
\t//}
    //2021/10/18 E

    //2021/10/18 S 新建築ポータル起動処理実装
    var strOpt = 'top=30,left=360,scrollbars=yes,resizable=yes,menubar=yes,location=yes,directories=yes,status=yes';
    //2023/02/22 S 建築ポータル起動条件変更対応
    //if(uid != '68199' && sec <= 11 && (jyu >= 5 && jyu <= 74) && ((shokushu == '100' && ccc == '3')||(shokushu == '101' && ccc == '3')|| shokushu == '130' ||  ▶
    if(uid != '68199' && sec <= 15 && ((jyu >= 5 && jyu <= 74) || jyu == 90 || jyu == 94 || jyu == 95) && ((shokushu == '100' && ccc == '3')||(shokushu == '101' ▶
        //2023/02/22 E
        //旧「できる！10%UP(さらに昔の生産性５％アップ)」起動条件と合致する場合は新建築ポータルを起動

        //2021/12/08 S
        //window.open('http://www.dho2.bc.obayashi.co.jp/','dho2',strOpt);
        //cookie値(dhostatus)が'1'でない場合、建築ポータルを起動。'1'の場合は起動しない
        if (strDhoStatus != '1'){window.open('http://www.dho2.bc.obayashi.co.jp/','dho2',strOpt);}
        //2021/12/08 E

\t} else {
        var blnChkTza01 = CheckTza01();
        if (blnChkTza01){
            //建築ポップアップの起動条件と合致する場合は新建築ポータルを起動

            //2021/12/08 S
            //window.open('http://www.dho2.bc.obayashi.co.jp/','dho2',strOpt);
            //cookie値(dhostatus)が'1'でない場合、建築ポータルを起動。'1'の場合は起動しない
            if (strDhoStatus != '1'){window.open('http://www.dho2.bc.obayashi.co.jp/','dho2',strOpt);}
            //2021/12/08 E
        }
    }
    //2021/10/18 E

\t//RNイントラPOPUP S
//2020/06/05 バイパスするよう設定
//    if (HttpRequest_Send("http://toraburun.fc.obayashi.co.jp/RNPAPI/API/RNPA001.aspx", null) == '1') {
    if (HttpRequest_Send("ReqTrans.aspx?p=http://toraburun.fc.obayashi.co.jp/RNPAPI/API/RNPA001.aspx", null) == '1') {
//2020/06/05
        //画面高さ取得
        var intWinH = window.screen.height;            
        //900を超えている場合は、高さ900とする。900未満の場合はタスクバー領域等を考慮し、少し縮めた高さに設定
        if (intWinH > 900) {intWinH = 900;}else{intWinH += -50;}
        //起動
        window.open("http://toraburun.fc.obayashi.co.jp/RNP/frm/RNPS001.aspx", "rnp", "width=576,height=" + intWinH + ",top=30,left=780,scrollbars=yes,resizable ▶
    }
\t//RNイントラPOPUP E

    //2023/12/05 トラブルＤＢ S
    if (HttpRequest_Send("ReqTrans.aspx?p=http://toraburun.fc.obayashi.co.jp/toradbapi/TDBPA001.aspx", null) == '1') {
        var subw = 632;
        var subh = 446;
        var subx = (window.screen.width - subw) / 2;
        var sbuy = (window.screen.height - subh) / 2;

        window.open("http://www.brc.osk.obayashi.co.jp/troubleDB/trial/troubldb_popup.html", "tdbp", "width=" + subw + ",height=" + subh + ",top=" + sbuy + ",le ▶

    }
    //2023/12/05 トラブルＤＢ E

    //2021/10/18 S 起動廃止
    //Exec_Tza01();
    //2021/10/18 S

\twindow.location.href = "http://www.fc.obayashi.co.jp";
\t
function getCookie(key){
    tmp=document.cookie+";";
    tmp1=tmp.indexOf(key, 0);
    if(tmp1!=-1){
        tmp=tmp.substring(tmp1, tmp.length);
        start=tmp.indexOf("=", 0)+ 1;
        end=tmp.indexOf(";", start);
        return(unescape(tmp.substring(start, end)));
    }
    return("");
}
//
// 文字列を右側から指定文字数だけ取り出す関数
//
function right( str, n ) {
    l = str.length;
    if (n>l) n=l;
    return( str.substring(l-n,l) );
}
//RNイントラPOPUP S
//通信処理
function HttpRequest_Send(strUrl, strParam) {
    var ajax = null;

    for ( i=0; i < 3; i++) {
        try {
            if (window.XMLHttpRequest) { ajax = new XMLHttpRequest(); }
            else if (window.ActiveXObject) {
                try {
                    ajax = new ActiveXObject("Msxml2.XMLHTTP");
                } catch (e) {
                    ajax = new ActiveXObject("Microsoft.XMLHTTP");
                }
            }
            //通信実行
            ajax.open("POST", strUrl, false);
            ajax.timeout = 5000;
            ajax.send(strParam);
            return ajax.responseText;

        } catch (e) {
            if(i == 2){
                return '0';
            }else{
                //alert(i + e.message);
                continue;
            }
        } finally {
            ajax = null;
        }
    }
}
//RNイントラPOPUP E

//2021/10/18 S 改修
//TZA01アクセス権限認証
function CheckTza01() {
    //参照先サーバ(★導入環境に合わせて設定変更)
    var strRootUrl = 'http://mieruka-popup.fc.obayashi.co.jp';

    //WebAPIに対するURL設定
    var strUrl = 'ReqTrans.aspx?p=' + strRootUrl + '/tza01/TZAA000/TZAA000';

    try {                
        //通信実行
        var objRet = sendQuery(strUrl, '');

        //結果確認(エラー、もしくはアクセス権なしの場合は処理抜ける)
        if (objRet == undefined || objRet == null) { return false; }

        var objRetJson = (new Function("return " + objRet))();

        if (objRetJson.ErrCode != 0 || objRetJson.RetCode != 1) { return false; }

        //認証ＯＫの場合はTrueを返却
        return true;

    } catch (e) {
        //処理を止めない為、スルー
        return false;
    }    
}
//2021/10/18 E

//通信(同期)
function sendQuery(send_url, send_data) {
    var ret_data;

    $.ajax({
        type: 'POST',
        async: false,
        url: send_url,
        data: send_data,
        success: function (data) {
            ret_data = data;
        }
    });
    return ret_data
}
//2020/05/28 E

</SCRIPT>
</HEAD>
<BODY>
<CENTER>
<H3>処理中です。しばらくお待ち下さい。</H3>
</CENTER>
</BODY>
</HTML>
"""


<?php

namespace App\Http\Controllers;

use GuzzleHttp\Client;
use Illuminate\Http\Request;

class ObayashiLoginController extends Controller
{
    public function login()
    {
        $client = new Client([
            'cookies' => true, // Enable cookie handling
            'headers' => [
                'User-Agent' => 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36',
                'Accept' => 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
                'Accept-Encoding' => 'gzip, deflate',
                'Accept-Language' => 'ja-JP,ja;q=0.9,en-US;q=0.8,en;q=0.7',
                'Cache-Control' => 'no-cache',
                'Connection' => 'keep-alive',
                'Origin' => 'http://login.fc.obayashi.co.jp',
                'Pragma' => 'no-cache',
                'Referer' => 'http://login.fc.obayashi.co.jp/sso/UI/Login?type=obayashi&goto=http%3A%2F%2Fintralogin.fc.obayashi.co.jp%2Fcgi-bin%2Fclogin.cgi%3Faccess%3Dhttp%253A%252F%252Fwww.fc.obayashi.co.jp%252F%252F',
                'Upgrade-Insecure-Requests' => '1',
            ],
        ]);

        $loginUrl = 'http://login.fc.obayashi.co.jp/sso/UI/Login';
        $successRedirectUrl = 'http://intralogin.fc.obayashi.co.jp/cgi-bin/clogin.cgi?access=http%3A%2F%2Fwww.fc.obayashi.co.jp%2F';

        $username = 'YOUR_USERNAME'; // Replace with the actual username
        $password = 'YOUR_PASSWORD'; // Replace with the actual password
        $groupCompanyCode = 'U'; // Replace with the desired group company code (e.g., 'U' for 大林組)

        try {
            // 1. Get the initial login page to potentially retrieve any necessary tokens or cookies
            $response = $client->get($loginUrl);
            $html = (string) $response->getBody();

            // You might need to parse the HTML here to extract any dynamic values
            // if the form has anti-CSRF tokens or other dynamic fields.
            // Based on the provided HTML, it doesn't seem to have any critical dynamic tokens.

            // 2. Prepare the login form data
            $loginData = [
                'IDToken0-0' => $groupCompanyCode,
                'IDToken0' => $groupCompanyCode, // Assuming the company code is the same as the group code initially
                'IDToken1' => $groupCompanyCode . $username, // Apply the JavaScript logic
                'IDToken2' => $password,
                'Login.Submit' => 'ログイン',
                'goto' => 'aHR0cDovL2ludHJhbG9naW4uZmMub2JheWFzaGkuY28uanAvY2dpLWJpbi9jbG9naW4uY2dpP2FjY2Vzcz1odHRwOi8vd3d3LmZjLm9iYXlhc2hpLmNvLmpwLw==',
                'gotoOnFail' => '',
                'SunQueryParamsString' => 'dHlwZT1vYmF5YXNoaQ==',
                'encoded' => 'true',
                'type' => 'obayashi',
                'gx_charset' => 'UTF-8',
                'IDButton' => 'ログイン', // Set the IDButton value as per the JavaScript
            ];

            // 3. Submit the login form
            $response = $client->post($loginUrl, [
                'form_params' => $loginData,
            ]);

            // 4. Check if the login was successful by inspecting the response and cookies
            // A successful login should result in a redirect to the success URL
            if ($response->getStatusCode() === 302 && $response->getHeaderLine('Location') === $successRedirectUrl) {
                // Optionally, you can make a subsequent request to the redirected URL
                $successPageResponse = $client->get($successRedirectUrl);
                $successPageContent = (string) $successPageResponse->getBody();

                // Do something with the successful login page content
                dd('Login successful!', $successPageContent);
            } else {
                // Login failed
                dd('Login failed.', (string) $response->getBody());
            }

        } catch (\GuzzleHttp\Exception\GuzzleException $e) {
            dd('An error occurred: ' . $e->getMessage());
        }
    }
}

![画像 (16)](https://github.com/user-attachments/assets/813622b4-a900-46cd-ab5d-ecc54c822c14)
![画像 (14)](https://github.com/user-attachments/assets/25ccd9e5-c8d6-4f79-8235-cbd264344471)
[login.txt](https://github.com/user-attachments/files/20218578/login.txt)
