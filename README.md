In the controller
class DLDWHDataImportController extends Controller
{

/*
開発者：ピョー
作成日：2024.10.08
概要　：DL・DWH情報データ取込
*/
protected $client;
protected $access_token;
protected $refresh_token;
protected $box_login_time;
protected $url_path;
protected $parentFolderId = "292962056862"; //本番フォルダー

public function __construct()
{
$this->client = new \GuzzleHttp\Client();
$this->client = new \GuzzleHttp\Client(['verify' => false]);
}
public function index()
{
return view('DLDWH.DLDWHDataImport');
}

/**
* ImportCsv
*
* Boxのデータ取込からジョブへdispatchし、結果返すまでの処理
*
* @param mixed $request
* @return mixed
*/
public function ImportCsv(Request $request)
{
$finishJob = [];
$loginUserName = $request->get('loginUserName');
if (session()->has('access_token')) {
$this->access_token = session('access_token');
if ($this->access_token == "") {
$finishJob = [
'token' => "no_token",
];
return response()->json($finishJob);
}

if (session()->has('box_login_time')) {
$this->box_login_time = session('box_login_time');
Log::info('box_login_time ' . $this->box_login_time);
}
if (session()->has('refresh_token')) {
$this->refresh_token = session('refresh_token');
Log::info('refresh_token ' . $this->refresh_token);
}
if (session()->has('url_path')) {
$this->url_path = session('url_path');
Log::info('url_path ' . $this->url_path);
}

$updatedDate = $this->getDateTime();
$updatedDateWithoutSeconds = Carbon::parse($updatedDate)->format('Y-m-d H:i');

//Boxからデータを取り込む
$DLDWHModel = new DLDHWDataImportModel();
$DLDWHModel->deleteFinshedJob();
$DLDWHModel->deleteDataImportLogs();
$DLDWHModel->deleteFailedJobs();
$DLDWHModel->deleteBoxExpiryStatus();
DLDWHJob::dispatch(
$this->parentFolderId,
$updatedDateWithoutSeconds,
$updatedDate,
$loginUserName,
$this->access_token,
$this->refresh_token,
$this->box_login_time,
$this->url_path
);

$finishJob = [
'isfinishJob' => false,
];
return response()->json($finishJob);
} else {
$finishJob = [
'token' => "no_token",
];
return response()->json($finishJob);
}
}


/**
* uploadDataToDB
*
* データベースへのデータ挿入処理を負う
*
* @param string $folderName
* @param mixed $allColumns
* @param mixed $batch
* @param boolean $isFileUpdated
*/
public function uploadDataToDB($folderName, $allColumns, $batch, $isFileUpdated)
{
$DLDWHModel = new DLDHWDataImportModel();
$result = $DLDWHModel->saveImportData($folderName, $allColumns, $batch, $isFileUpdated);
return $result;
}

/**
* checkJobStatus
*
* ジョブ処理によってデータベースに全データ取り込んだか確認をする
*
* @param mixed $request
* @return mixed
*/
public function checkJobStatus(Request $request)
{
Log::info('start checkJobStatus');
$DLDWHModel = new DLDHWDataImportModel();
$job = $DLDWHModel->getFinishedJob();
$failedJob = $DLDWHModel->getFailedJobs();

if ($job) {
$data = $job ? json_decode($job->logs, true) : null;
if ($data && in_array('no_upload_file', $data)) {
return response()->json([
'status' => 'no_upload_file'
]);
} else {
return response()->json([
'status' => 'completed',
'data' => $data
]);
}
} elseif ($failedJob) {
return response()->json([
'status' => 'failedJob',
'data' => $failedJob ? $failedJob->exception : null
]);
} else {
return response()->json([
'status' => 'pending'
]);
}
}

/**
* LoadDataFromBox
*
* BoxからCSVデータをフォルダー名ごとに整理して抽出する
*
* @param string $folderId
* @param string $updatedDateWithoutSeconds
* @param string $updatedDate
* @param string $access_token
* @return mixed
*/
public function LoadDataFromBox($folderId, $updatedDateWithoutSeconds, $updatedDate, $access_token)
{
$folderItems = $this->getFolderItems($folderId, $access_token);
$csvFiles = [];
$folderName = "";

foreach ($folderItems as $item) {
if ($item['type'] === "folder") {
$folderName = $item['name'];
$childCsvFiles = $this->LoadDataFromBox($item['id'], $updatedDateWithoutSeconds, $updatedDate, $access_token);
if (!empty($childCsvFiles)) {
$csvFiles[$folderName] = $childCsvFiles;
}
} elseif ($item['type'] === "file" && pathinfo($item['name'], PATHINFO_EXTENSION) === 'csv') {
if (!isset($csvFiles[$folderName])) {
$csvFiles[$folderName] = [];
}
$csvFiles[$folderName][] = $item; // 配列に追加
}
}

// フォルダ内のファイルをフォルダー名ごとで整理する
$sortedCsvFiles = [];
foreach ($csvFiles as $folderName => $items) {
if (!empty($items)) {
$sortedCsvFiles[$folderName] = $items;
}
}
return $sortedCsvFiles;
}

/**
* copyDataElementListLevel
*
* copyData from doc_0201要素リスト to Doc_0201要素リスト_レベル
* @param string $tableName
* @param string $latestUpdateDate
* @return void
*/
function copyDataElementListLevel($tableName, $latestUpdateDate)
{
$DLDWHModel = new DLDHWDataImportModel();
$tableExit = $DLDWHModel->checkTableExist($tableName);
$subTable = "doc_0201要素リスト_レベル";
if ($tableExit) {
$DLDWHModel->copyDataElementListLevel(strtolower($tableName), $subTable, $latestUpdateDate);
}
}

/**
* getFolderItems
*
* フォルダー情報を取得する
*
* @param string $folderId
* @param string $access_token
* @return array
*/
public function getFolderItems($folderId, $access_token)
{
$requestURL = "https://api.box.com/2.0/folders/" . $folderId . "/items/";
$header = [
"Authorization" => "Bearer " . $access_token,
"Accept" => "application/json"
];

$allItems = [];
$offset = 0;
$limit = 100;

try {
do {
$params = [
'headers' => $header,
'query' => [
'limit' => $limit,
'offset' => $offset
]
];

$response = $this->client->request('GET', $requestURL, $params);
$folderItems = json_decode($response->getBody()->getContents(), true);

if (!isset($folderItems['entries'])) {
throw new Exception("Invalid response structure: 'entries' key not found.");
}

$allItems = array_merge($allItems, $folderItems['entries']);
$offset += $limit;

// log::info("Fetched " . count($folderItems['entries']) . " items, current offset: " . $offset);
} while (count($folderItems['entries']) == $limit);

// log::info("Total items fetched: " . count($allItems));
return $allItems;
} catch (Exception $e) {
log::error("Error fetching folder items: " . $e->getMessage());
return response()->json(['error' => $e->getMessage()], 500)->json();
}
}

/**
* getUpdateHistory
*
* 取込履歴を取得する
*
* @return mixed
*/
function getUpdateHistory()
{
$DLDWHModel = new DLDHWDataImportModel();
$updateHistory = $DLDWHModel->getUpdateHistory();
return $updateHistory;
}

/**
* getDateTime
*
* @return string
*/
public function getDateTime()
{
// 現在の日付と時刻を取得
$now = Carbon::now(); // Carbonを使用して現在の日時を取得

// フォーマットを指定
$formattedDateTime = $now->format('Y-m-d H:i:s');

// フォーマットされた日付を表示
return $formattedDateTime;
}

/**
* deleteData
*
* 全データを削除する
*
* @return mixed
*/
public function deleteData()
{
set_time_limit(300);
$DLDWHModel = new DLDHWDataImportModel();
$updateHistory = $DLDWHModel->deleteData();
return $updateHistory;
}

/**
* saveAccessToken
*
* BOXログイン有効時間切れた後、更新したアクセストーケンを再保存する
*
* @param string $accessToken
* @param string $refreshToken
* @return void
*/
public function saveAccessToken($accessToken, $refreshToken)
{
session()->forget('access_token');
session()->forget('refresh_token');
session(['access_token' => $accessToken]);
session(['refresh_token' => $refreshToken]);
}
}


In the job

class DLDWHJob implements ShouldQueue
{
use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
/**
* Create a new job instance.
*
* @return void
*/
protected $parentFolderId;
protected $updatedDateWithoutSeconds;
protected $updatedDate;
protected $loginUserName;
protected $fileNameList;
protected $folderNameList;
protected $access_token;
protected $refresh_token;
protected $box_login_time;
protected $url_path;
protected $client;
protected $DOC_0201_ELEMENT_LISTS = "doc_0201要素リスト";
protected $COPY_DOC_0211_PROFILE_LISTS = "doc_0211プロファイルリスト";
protected $COPY_DOC_0201_ELEMENT_LEVEL_LISTS = "doc_0201要素リスト_レベル";
protected $COPY_DOC_0201_ELEMENT_NEST_LISTS = "doc_0211要素リスト_ネスト";
protected $COPY_DOC_0201_ELEMENT_Kosei_LISTS = "doc_0211要素リスト_構成要素";
protected $COPY_Para_0101_DoorWindow_Quantity = "para_0112インスタンスパラ_ドア窓個数";
protected $COPY_Para_0201_DoorWindow_Quantity = "para_0211タイプパラ_ドア窓個数";
protected $Int_Target_Category_Info = "int_0106対象カテゴリ情報";
protected $doc_0301_phase = "doc_0301フェーズ";
protected $COPY_doc_0301_KouchikuPhase = "doc_0301フェーズ_構築フェーズ";
protected $COPY_doc_0301_Kaitai_Phase = "doc_0301フェーズ_解体フェーズ";
protected $Copy_Int_Target_Category_Info = "int_0106対象カテゴリ情報_ルート解析";

protected $COPY_Para_0101_WorkSet = "para_0111インスタンスパラ_ワークセット";

protected $COPY_Para_0101_DesignOption = "para_0111インスタンスパラ_デザインオプション";
protected $COPY_Para_0101_Phase = "para_0111インスタンスパラ_フェーズ";

public function __construct(
$parentFolderId = null,
$updatedDateWithoutSeconds = null,
$updatedDate = null,
$loginUserName = null,
$access_token = null,
$refresh_token = null,
$box_login_time = null,
$url_path = null
) {
$this->parentFolderId = $parentFolderId;
$this->updatedDateWithoutSeconds = $updatedDateWithoutSeconds;
$this->updatedDate = $updatedDate;
$this->loginUserName = $loginUserName;
$this->access_token = $access_token;
$this->refresh_token = $refresh_token;
$this->box_login_time = $box_login_time;
$this->url_path = $url_path;
}

/**
* Execute the job.
*
* @return void
*/
public function handle()
{
$this->client = new \GuzzleHttp\Client();
$this->client = new \GuzzleHttp\Client(['verify' => false]);
$DLDWHModel = new DLDHWDataImportModel();
$dldwhImportController = new DLDWHDataImportController();

try {
Log::info('start LoadDataFromBox ');
$csvFiles = $dldwhImportController->LoadDataFromBox(
$this->parentFolderId,
$this->updatedDateWithoutSeconds,
$this->updatedDate,
$this->access_token
);

if (!empty($csvFiles)) {
Log::info('start organizeData ' . count($csvFiles));
$latestUpdateDate = $DLDWHModel->getLatestUpdatedDate_document();
$latestDate_KoSuu = $DLDWHModel->getLatestDateForKoSuu();
$this->organizeAndProcessData($csvFiles);

// copyData from Para_0101ドア窓*** to Para_0112インスタンスパラ_ドア窓個数
$DLDWHModel->copyPara0101DataList("Para_0101", $this->COPY_Para_0101_DoorWindow_Quantity, $latestUpdateDate);

// 最終取込履歴の保存
$DLDWHModel->saveUpdateHistory($this->loginUserName, $this->updatedDate);

// ジョブ完了の保存
$successMessage = [];
$jsonResultList = $DLDWHModel->getDataImportLogs();
foreach ($jsonResultList as $result) {
array_push($successMessage, json_decode($result->import_logs, true)); // trueを追加して配列としてデコード
}
$finishJob = [
'fileNames' => $this->fileNameList,
'folderNames' => $this->folderNameList,
'successMessage' => $successMessage,
];
$jsonFinishJob = json_encode($finishJob);

$DLDWHModel->saveFinishedJob($jsonFinishJob);
} else {
//取り込むファイルがない
$finishJob = ["no_upload_file"];
$jsonFinishJob = json_encode($finishJob);
$DLDWHModel->saveFinishedJob($jsonFinishJob);
}
} catch (Exception $e) {
Log::error('DLDWHJob failed: ' . $e->getMessage(), [
'file' => $e->getFile(),
'line' => $e->getLine(),
'trace' => $e->getTraceAsString()
]);
$this->fail($e);
}
}

/**
* organizeAndProcessData
* BOXからデータを取得し、データベースに保存する
*
* @param mixed $folderItems
*
* @return void
*/
public function organizeAndProcessData($folderItems)
{
$DLDWHModel = new DLDHWDataImportModel();
$dldwhImportController = new DLDWHDataImportController();
$sortedTableNames = $DLDWHModel->showDependencyGraph();

$normalizedFolderItems = [];
foreach ($folderItems as $key => $value) {
// $normalizedFolderItems[strtolower($key)] = $value;
$normalizedFolderItems[mb_strtolower($key, 'UTF-8')] = $value;
}

foreach ($sortedTableNames as $tableName) {
if (!isset($normalizedFolderItems[$tableName])) {
Log::info("Organize Data Skip tables>>>>>" . $tableName);
continue;
}
$csvFiles = $normalizedFolderItems[$tableName];
$tableExit = $DLDWHModel->checkTableExist($tableName);
$allColumns = $DLDWHModel->getTableColumns($tableName);

if ($tableExit) {
foreach ($csvFiles as $files) {
foreach ($files as $file) {
$fileName = $file['name'];
$fileId = $file['id'];
$isFileUpdated = false;
$successMessage = [];

$retryCount = 0;
$maxRetries = 3;
$boxDate = "";

$this->checkBoxExpiryStatus();
log::info(" this->access_token >>>>>>>>>>>>>>>>>>>>>" . $this->access_token);
while ($retryCount < $maxRetries) {
    try {
    $boxDate=$this->getDateOrNameOfCsv($fileId, "date", $this->access_token); //当csvのboxからの日付を取得
    break; // 成功した場合はループを抜ける
    } catch (Exception $e) {
    $retryCount++;
    sleep(5); // 5秒待機して再試行
    }
    }

    if (empty($folderItems)) {
    throw new Exception("Failed to retrieve folder items after $maxRetries retries.");
    }

    $boxDateParsed = Carbon::parse($boxDate);
    $csvFileDate = $DLDWHModel->getCsvFileDate($fileName);

    if (!is_null($csvFileDate)) {
    $lastModifiedAt = $csvFileDate->last_modified_at;
    $fileDateParsed = Carbon::parse($lastModifiedAt);
    if ($boxDateParsed > $fileDateParsed) {
    $isFileUpdated = true;
    }
    }

    if (is_null($csvFileDate) || $isFileUpdated) {
    $requestURL = "https://api.box.com/2.0/files/" . $fileId . "/content";
    $header = [
    "Authorization" => "Bearer " . $this->access_token,
    "Accept" => "application/json"
    ];
    $response = $this->client->request('GET', $requestURL, ['headers' => $header]);
    $stream = $response->getBody()->detach();

    $firstLine = fgets($stream);
    if (substr($firstLine, 0, 3) === "\xEF\xBB\xBF") {
    $firstLine = substr($firstLine, 3);
    }

    $encoding = mb_detect_encoding($firstLine, ['UTF-8', 'SJIS', 'EUC-JP', 'ISO-2022-JP']);
    $firstLine = mb_convert_encoding($firstLine, 'UTF-8', $encoding);
    $header = array_map('trim', explode(',', $firstLine));

    if (is_array($this->fileNameList) && is_array($this->folderNameList)) {
    array_push($this->fileNameList, $fileName);
    array_push($this->folderNameList, $tableName);
    } else {
    $this->fileNameList = [$fileName];
    $this->folderNameList = [$tableName];
    }

    $batchData = [];
    log::info("FIle Name >>>>>>>>>>>>>>>>>>>>>" . $fileName);
    while (($line = fgets($stream)) !== false) {
    $line = mb_convert_encoding($line, 'UTF-8', $encoding);
    // $row = str_getcsv($line);
    $row = explode(',', $line);

    if (count($header) !== count($row)) {
    // Remove the last column if header count is less than row count and the last value contains \r\n
    if (count($header) < count($row) && strpos(end($row), "\r\n" ) !==false) {
        array_pop($row);
        }

        // Check if the last value of the row contains \r\n
        if (strpos(end($row), "\r\n" ) !==false) {
        // Remove \r\n from the last value
        $row[count($row) - 1]=str_replace("\r\n", "" , end($row));
        }

        // Add empty strings until the row count matches the header count
        while (count($row) < count($header)) {
        $row[]="" ;
        }
        }

        $rowData=array_combine($header, $row);
        $tableRow=[];
        foreach ($allColumns as $column) {
        $tableRow[$column]=$column==='日付' ? $this->updatedDateWithoutSeconds : ($rowData[$column] ?? null);
        }
        $batchData[] = $tableRow;

        if (count($batchData) >= 3000) { // Adjust batch size as needed
        // $result = $dldwhImportController->uploadDataToDB($tableName, $allColumns, $batchData, $isFileUpdated);
        $result = $DLDWHModel->saveImportData($tableName, $allColumns, $batchData, $isFileUpdated);
        array_push($successMessage, $result);
        $batchData = [];
        }
        }

        if (!empty($batchData)) {
        // $result = $dldwhImportController->uploadDataToDB($tableName, $allColumns, $batchData, $isFileUpdated);
        $result = $DLDWHModel->saveImportData($tableName, $allColumns, $batchData, $isFileUpdated);
        array_push($successMessage, $result);
        }
        $this->processResult($result, $tableName, $fileName, $DLDWHModel);

        fclose($stream);
        $DLDWHModel->saveCsvFileDate($fileName, $this->updatedDate, $isFileUpdated);
        }
        }
        }
        switch ($tableName) {
        case $this->DOC_0201_ELEMENT_LISTS:
        $latestUpdateDate = $DLDWHModel->getLatestUpdatedDate_element();
        $DLDWHModel->copyDataProfileList($this->DOC_0201_ELEMENT_LISTS, $this->COPY_DOC_0211_PROFILE_LISTS, $latestUpdateDate);
        $DLDWHModel->copyDataElementList($this->DOC_0201_ELEMENT_LISTS, $this->COPY_DOC_0201_ELEMENT_LEVEL_LISTS, $latestUpdateDate);
        $DLDWHModel->copyDataNestingList($this->DOC_0201_ELEMENT_LISTS, $this->COPY_DOC_0201_ELEMENT_NEST_LISTS, $latestUpdateDate);
        $DLDWHModel->copyDataKoseiList($this->DOC_0201_ELEMENT_LISTS, $this->COPY_DOC_0201_ELEMENT_Kosei_LISTS, $latestUpdateDate);
        break;
        case $this->Int_Target_Category_Info:
        $latestUpdateDate = $DLDWHModel->getLatestUpdatedDate_Int_target_categoty_info();
        $DLDWHModel->copyInt_target_categoty_infoList($this->Int_Target_Category_Info, $this->Copy_Int_Target_Category_Info, $latestUpdateDate);
        break;
        case $this->doc_0301_phase:
        $latestUpdateDate = $DLDWHModel->getLatestUpdatedDate_doc_0301_phase();
        $DLDWHModel->copyKouchikuPhase($this->doc_0301_phase, $this->COPY_doc_0301_KouchikuPhase, $latestUpdateDate);
        $DLDWHModel->copyKaitaiPhase($this->doc_0301_phase, $this->COPY_doc_0301_Kaitai_Phase, $latestUpdateDate);
        break;
        }
        }
        }
        }

        /**
        * processResult
        * 取込ログを保存する
        *
        * @param mixed $successMessage
        * @param string $folderName
        * @param string $fileName
        * @param DLDHWDataImportModel $DLDWHModel
        *
        * @return void
        */
        function processResult($successMessage, $folderName, $fileName, $DLDWHModel)
        {
        $duplicates = [];
        foreach ($successMessage as $sm) {
        $combinedKey = $folderName . '/' . $fileName; // Combine folderName and fileName
        if (!isset($duplicates[$combinedKey])) {
        $duplicates[$combinedKey] = [];
        }
        if (isset($sm['success']) && $sm['success'] === 'duplicate') {
        $duplicates[$combinedKey][] = $sm['duplicateRows'];
        } else {
        $duplicates[$combinedKey][] = ["OK"];
        }
        }
        if (isset($duplicates)) {
        $jsonResult = json_encode($duplicates);
        $DLDWHModel->saveDataImportLogs($jsonResult);
        }
        }

        /**
        * getDateOrNameOfCsv
        * csvファイルの日付または名前を取得する
        *
        * @param string $fileId
        * @param string $DateOrName
        * @param string $access_token
        * @return string
        */
        function getDateOrNameOfCsv($fileId, $DateOrName, $access_token)
        {

        $result = '';
        $requestURL = "https://api.box.com/2.0/files/" . $fileId;
        $header = [
        "Authorization" => "Bearer " . $access_token,
        "Accept" => "application/json"
        ];
        $response = $this->client->request('GET', $requestURL, ['headers' => $header]);
        $files = json_decode($response->getBody()->getContents(), true);
        if ($DateOrName === "date") {
        $date = $files["modified_at"];

        // Create a DateTime object from the modified_at date
        $dateTime = new \DateTime($date, new \DateTimeZone('UTC'));

        // Convert the DateTime object to Japan's timezone
        $dateTime->setTimezone(new \DateTimeZone('Asia/Tokyo'));

        // データベース用のフォーマットに変換
        $result = $dateTime->format('Y-m-d H:i:s');
        } else {
        $result = $files["name"];
        }

        return $result;
        }

        /**
        * checkBoxExpiryStatus
        * BOXログイン有効時間を確認し、切れる恐れがある場合、トーケンを更新する
        *
        * @return void
        */
        public function checkBoxExpiryStatus()
        {
        $dldwhImportController = new DLDWHDataImportController();
        $tokenExpiryTime = $this->box_login_time + 3600; //box期限60分
        $clientId = "";
        $secrectKey = "";
        $accessToken = "";

        if (strpos($this->url_path, 'deployment') !== false) {
        $clientId = env("BOX_CLIENT_ID_FOR_DEV_SLOT");
        $secrectKey = env("BOX_CLIENT_SECRET_FOR_DEV_SLOT");
        } else {
        $clientId = env("BOX_CLIENT_ID");
        $secrectKey = env("BOX_CLIENT_SECRET");
        }

        // トークンの有効期限が10分未満の場合、新しいトークンを取得
        if (time() > $tokenExpiryTime - 600) { // 10分未満の場合
        Log::info('Token is about to expire');
        $response = $this->client->post('https://api.box.com/oauth2/token', [
        'form_params' => [
        'grant_type' => 'refresh_token',
        'client_id' => $clientId,
        'client_secret' => $secrectKey,
        'refresh_token' => $this->refresh_token,
        ],
        ]);

        $data = json_decode($response->getBody(), true);
        $accessToken = $data['access_token'];
        $refreshToken = $data['refresh_token'];
        $this->refresh_token = $refreshToken;
        $this->access_token = $accessToken;

        $dldwhImportController->saveAccessToken($accessToken, $refreshToken);
        Log::info('accessToken : ' . $accessToken);
        Log::info('refreshToken : ' . $refreshToken);

        $this->saveBoxExpiryStatus();
        $this->box_login_time = time();
        }
        }

        /**
        * saveBoxExpiryStatus
        * トーケンを更新した後、記録する
        *
        * @return void
        */
        public function saveBoxExpiryStatus()
        {
        $query = "INSERT INTO box_expiry_status (status) VALUES (?)";
        $result = DB::connection('dldwh')->insert($query, ['true']);
        }


        // ジョブの再試行回数とタイムアウトを設定
        public $tries = 3; // 最大試行回数
        }
