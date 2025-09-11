CREATE TABLE `model_file_cache` (
  `id` bigint(20) UNSIGNED NOT NULL AUTO_INCREMENT,
  `project_box_id` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
  `file_name` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
  `base_name` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
  `box_file_id` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
  `file_type` varchar(10) COLLATE utf8mb4_unicode_ci NOT NULL,
  `content` longtext COLLATE utf8mb4_unicode_ci,
  `box_modified_at` datetime NOT NULL,
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `box_file_id_unique` (`box_file_id`),
  KEY `project_box_id_index` (`project_box_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;




<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\DB;
use GuzzleHttp\Client; // Guzzleをインポート
use Exception;

class DLDHWDataImportModel extends Model
{
    // ... 既存のgetCategoryNameByElementIdメソッドなど ...

    /**
     * 【新規】Boxにログインしていない時に、DBキャッシュからモデルデータを取得する
     */
    public function getCachedModelData($projectFolderId)
    {
        $cachedFiles = DB::table('model_file_cache')
                         ->where('project_box_id', $projectFolderId)
                         ->get()
                         ->groupBy('base_name');

        if ($cachedFiles->isEmpty()) {
            return ['error' => 'No cached model found. Please log in to Box to sync data for the first time.'];
        }
        
        return $this->formatDataForFrontend($cachedFiles);
    }

    /**
     * 【新規】Boxにログインしている時に、BoxとDBを同期し、最新のモデルデータを取得する
     */
    public function syncAndGetModelData($projectFolderId)
    {
        try {
            // 1. Box APIから最新のファイルリストを取得
            $boxFiles = $this->fetchFullBoxFileList($projectFolderId);

            // 2. DBから現在のファイルリストを取得
            $dbFiles = DB::table('model_file_cache')
                         ->where('project_box_id', $projectFolderId)
                         ->get()
                         ->keyBy('box_file_id');

            $boxFileIds = collect($boxFiles)->pluck('id')->all();

            // 3. Boxの最新情報とDBを比較し、更新/追加
            foreach ($boxFiles as $boxFile) {
                $dbFile = $dbFiles->get($boxFile->id);

                if (!$dbFile || new \DateTime($dbFile->box_modified_at) < new \DateTime($boxFile->modified_at)) {
                    // DBにない、またはBoxの方が新しい場合はファイルをダウンロードしてDBを更新
                    $content = $this->fetchBoxFileContent($boxFile->id);
                    
                    DB::table('model_file_cache')->updateOrInsert(
                        ['box_file_id' => $boxFile->id], // このIDで検索
                        [ // 以下の内容で更新または作成
                            'project_box_id' => $projectFolderId,
                            'file_name' => $boxFile->name,
                            'base_name' => pathinfo($boxFile->name, PATHINFO_FILENAME),
                            'file_type' => strtolower(pathinfo($boxFile->name, PATHINFO_EXTENSION)),
                            'content' => $content,
                            'box_modified_at' => new \DateTime($boxFile->modified_at),
                            'created_at' => now(),
                            'updated_at' => now(),
                        ]
                    );
                }
            }
            
            // 4. Boxで削除されたファイルをDBから削除
            $dbFileIds = $dbFiles->pluck('box_file_id')->all();
            $deletedIds = array_diff($dbFileIds, $boxFileIds);
            if (!empty($deletedIds)) {
                DB::table('model_file_cache')->whereIn('box_file_id', $deletedIds)->delete();
            }

        } catch (Exception $e) {
            // Boxとの通信に失敗した場合は、エラーをログに記録し、既存のキャッシュを返す
            Log::error("Box Sync Failed: " . $e->getMessage());
        }
        
        // 同期後、DBから最新のデータを取得して返す
        return $this->getCachedModelData($projectFolderId);
    }
    
    /**
     * 【新規・ヘルパー】フロントエンド用にデータを整形する
     */
    private function formatDataForFrontend($groupedFiles)
    {
        $objMtlPairs = [];
        foreach ($groupedFiles as $baseName => $files) {
            $objFile = $files->firstWhere('file_type', 'obj');
            $mtlFile = $files->firstWhere('file_type', 'mtl');

            if ($objFile) {
                $objMtlPairs[] = [
                    'baseName' => $baseName,
                    'obj' => ['name' => $objFile->file_name, 'content' => $objFile->content],
                    'mtl' => $mtlFile ? ['name' => $mtlFile->file_name, 'content' => $mtlFile->content] : null
                ];
            }
        }
        return $objMtlPairs;
    }

    /**
     * 【新規・ヘルパー】Boxからファイルリストを全て取得（更新日時も含む）
     */
    private function fetchFullBoxFileList($projectFolderId)
    {
        $accessToken = session('access_token');
        $client = new Client(['verify' => false]);
        $header = ["Authorization" => "Bearer " . $accessToken];
        $allFolderItems = [];
        $offset = 0;
        $limit = 1000;

        do {
            // "modified_at"フィールドをリクエストに追加
            $requestURL = "https://api.box.com/2.0/folders/{$projectFolderId}/items?fields=id,name,modified_at&limit={$limit}&offset={$offset}";
            $response = $client->get($requestURL, ['headers' => $header]);
            $body = json_decode($response->getBody()->getContents());
            
            if (isset($body->entries)) {
                $allFolderItems = array_merge($allFolderItems, $body->entries);
                $offset += count($body->entries);
            } else {
                break;
            }
        } while ($offset < $body->total_count);

        return $allFolderItems;
    }

    /**
     * 【新規・ヘルパー】Boxから単一のファイルの中身を取得
     */
    private function fetchBoxFileContent($fileId)
    {
        $accessToken = session('access_token');
        $client = new Client(['verify' => false]);
        $header = ["Authorization" => "Bearer " . $accessToken];
        $requestURL = "https://api.box.com/2.0/files/{$fileId}/content";
        
        $response = $client->get($requestURL, ['headers' => $header]);
        
        return $response->getBody()->getContents();
    }
}





<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log; // Logファサードをインポート
use Exception;

class DLDWHDataObjectViewerController extends Controller
{
    public function objViewer()
    {
        return view('DLDWH.OBJViewer');
    }

    /**
     * 【新規】モデルデータを取得するための唯一のエンドポイント
     * Boxログイン状態に応じて、キャッシュまたは同期後のデータを返す
     */
    public function getModelData(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        if (empty($projectFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }

        $dldwhModel = new DLDHWDataImportModel();
        
        // Boxにログインしているかチェック
        if (session()->has('access_token') && !empty(session('access_token'))) {
            // ログイン時：Boxと同期し、最新データを返す
            $data = $dldwhModel->syncAndGetModelData($projectFolderId);
        } else {
            // 未ログイン時：DBキャッシュからデータを返す
            $data = $dldwhModel->getCachedModelData($projectFolderId);
        }

        if (isset($data['error'])) {
            return response()->json($data, 404);
        }

        return response()->json($data);
    }

    /**
     * この関数は変更なし
     */
    public function getCategoryNameByElementId(Request $request)
    {
        $WSCenID = $request->input('WSCenID');
        $elementIds = $request->input('ElementIds');
        $dldwhModle = new DLDHWDataImportModel();
        return $dldwhModle->getCategoryNameByElementId($WSCenID, $elementIds);
    }
    
    /**
     * この関数は変更なし
     */
    public function getProjectList(Request $request)
    {
        // ... 既存のコード ...
    }


    /*
     * 以下の関数は新しいアーキテクチャでは不要になります。
     * コメントアウトまたは削除してください。
     *
    public function getObjList(Request $request) { ... }
    public function getDownloadUrls(Request $request) { ... }
    */
}



import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/controls/OBJLoader.js';
import { MTLLoader } from './library/controls/MTLLoader.js';

// --- Get UI Elements ---
const loaderContainer = document.getElementById('loader-container');
const loaderTextElement = document.getElementById('loader-text');
const modelTreePanel = document.getElementById('modelTreePanel');
const modelTreeList = document.getElementById('modelTreeList');
const modelTreeSearch = document.getElementById('modelTreeSearch');
const closeModelTreeBtn = document.getElementById('closeModelTreeBtn');
const objectInfoPanel = document.getElementById('objectInfo');
const modelSelector = document.getElementById('model-selector');
const viewerContainer = document.getElementById('viewer-container');
const toggleUiButton = document.getElementById('toggle-ui-button');
const resetViewButton = document.getElementById('resetViewButton');

// --- Global variables ---
let parsedWSCenID = "";
let parsedPJNo = "";
let loadedObjectModelRoot = null;
let initialCameraPosition = new THREE.Vector3(10, 10, 10);
let initialCameraLookAt = new THREE.Vector3(0, 0, 0);
let selectedObjectOrGroup = null;
const originalMeshMaterials = new Map();
const originalObjectPropertiesForIsolate = new Map();
let isIsolateModeActive = false;
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
let mouseDownPosition = new THREE.Vector2();
const highlightColorSingle = new THREE.Color(0xa0c4ff);
const elementIdDataMap = new Map();

const BOX_MAIN_FOLDER_ID = "339110566808";
var CSRF_TOKEN = $('meta[name="csrf-token"]').attr('content');

// --- Scene Setup ---
const scene = new THREE.Scene();
const backgroundGeometry = new THREE.PlaneGeometry(2, 2, 1, 1);
const backgroundMaterial = new THREE.ShaderMaterial({
    vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = vec4(position.xy, 1.0, 1.0); }`,
    fragmentShader: `uniform vec3 topColor; uniform vec3 bottomColor; varying vec2 vUv; void main() { gl_FragColor = vec4(mix(bottomColor, topColor, vUv.y), 1.0); }`,
    uniforms: { topColor: { value: new THREE.Color(0xd8e3ee) }, bottomColor: { value: new THREE.Color(0xf0f0f0) } },
    depthWrite: false
});
const backgroundMesh = new THREE.Mesh(backgroundGeometry, backgroundMaterial);
backgroundMesh.renderOrder = -1;
scene.add(backgroundMesh);
const camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 0.1, 20000);
camera.position.copy(initialCameraPosition);
camera.lookAt(initialCameraLookAt);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
viewerContainer.appendChild(renderer.domElement);
const ambientLight = new THREE.AmbientLight(0x606060, 2);
scene.add(ambientLight);
const directionalLight = new THREE.DirectionalLight(0xFFFFFF, 2.5);
directionalLight.position.set(50, 100, 75);
directionalLight.castShadow = true;
scene.add(directionalLight);
const hemiLight = new THREE.HemisphereLight(0xffffff, 0x8d8d8d, 1.5);
hemiLight.position.set(0, 50, 0);
scene.add(hemiLight);
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;

// --- Main Application Logic ---
$(document).ready(function () {
    const login_user_id = $("#hidLoginID").val();
    if (typeof recordAccessHistory === 'function') {
        recordAccessHistory(login_user_id, "/DL_DWH.png", "DLDWH/objviewer", "OBJビューア");
    }
    onWindowResize();
    populateProjectDropdown();
    animate();
    if (resetViewButton) {
        resetViewButton.addEventListener('click', handleResetView);
    }
});

async function populateProjectDropdown() {
    try {
        const projects = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getProjectList",
            data: { _token: CSRF_TOKEN, folderId: BOX_MAIN_FOLDER_ID }
        });
        modelSelector.innerHTML = '';
        if (projects.includes("no_token")) {
            // Boxにログインしていなくても、キャッシュがあれば動作するのでアラートは出さない
        } 
        
        if (projects && projects.length > 0) {
            projects.forEach(project => {
                const option = document.createElement('option');
                option.value = project.id;
                option.textContent = project.name;
                modelSelector.appendChild(option);
            });
            const modelToLoad = projects.length > 1 ? projects[1] : projects[0];
            loadModel(modelToLoad.id, modelToLoad.name);
        } else {
            if (loaderTextElement) loaderTextElement.textContent = "No projects found.";
        }
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "Error fetching project list.";
    }
}

/**
 * **データベースキャッシュを利用する新しいロード関数**
 */
async function loadModel(projectFolderId, projectName) {
    resetScene();
    if (loaderContainer) loaderContainer.style.display = 'flex';

    try {
        // --- STEP 1: サーバーからファイルの中身を直接取得 ---
        if (loaderTextElement) loaderTextElement.textContent = `Fetching model data for ${projectName}...`;
        const filePairs = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getModelData", // 新しいエンドポイント
            data: { _token: CSRF_TOKEN, folderId: projectFolderId }
        });

        if (!Array.isArray(filePairs) || filePairs.length === 0) {
            throw new Error(`No model data found for project "${projectName}".`);
        }
        
        if (loaderTextElement) loaderTextElement.textContent = `Processing geometry and materials...`;

        // --- STEP 2: 各ファイルペアを解析 ---
        const allLoadedObjects = [];
        const firstObjContent = filePairs.find(p => p.obj)?.obj.content;
        if (firstObjContent) {
            const headerData = await parseObjHeader(firstObjContent);
            if (headerData) {
                parsedWSCenID = headerData.wscenId;
                parsedPJNo = headerData.pjNo;
            }
        }
        
        for (const pair of filePairs) {
            try {
                const objContent = pair.obj.content;
                if (!objContent || !objContent.trim()) continue;

                let materialsCreator = null;
                if (pair.mtl && pair.mtl.content && pair.mtl.content.trim()) {
                    materialsCreator = new MTLLoader().parse(pair.mtl.content, '');
                }

                const objLoader = new OBJLoader();
                if (materialsCreator) objLoader.setMaterials(materialsCreator);
                
                const parsedGroup = objLoader.parse(objContent);
                if (parsedGroup) allLoadedObjects.push(parsedGroup);

            } catch (e) {
                console.error("Could not parse OBJ/MTL pair:", { name: pair.baseName, error: e });
            }
        }

        // --- STEP 3: ジオメトリを検証し、結合 ---
        loadedObjectModelRoot = new THREE.Group();
        const validationBox = new THREE.Box3();

        allLoadedObjects.forEach(parsedGroup => {
            if (parsedGroup && parsedGroup.isGroup) {
                const childrenToAdd = [];
                parsedGroup.children.forEach(child => {
                    if (child.isMesh && child.geometry) {
                        try {
                            validationBox.setFromObject(child);
                            childrenToAdd.push(child);
                        } catch (e) {
                            console.warn("Discarding object with invalid geometry:", { name: child.name, error: e });
                        }
                    }
                });
                loadedObjectModelRoot.add(...childrenToAdd);
            }
        });
        
        if (loadedObjectModelRoot.children.length === 0) {
             throw new Error("No valid objects could be loaded. Files may be empty or corrupted.");
        }

        // --- STEP 4: モデル全体の位置調整とスケーリング ---
        const box = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        const center = box.getCenter(new THREE.Vector3());
        loadedObjectModelRoot.position.sub(center);
        const size = box.getSize(new THREE.Vector3());
        const maxDim = Math.max(size.x, size.y, size.z);
        const scale = (maxDim > 0) ? (150 / maxDim) : 1;
        loadedObjectModelRoot.scale.set(scale, scale, scale);
        loadedObjectModelRoot.userData.modelScale = scale;
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;

        // --- STEP 5: カテゴリデータを取得 ---
        const allIds = [...new Set(
            loadedObjectModelRoot.children
                .map(child => {
                    const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                    return splitIndex > 0 ? child.name.substring(splitIndex + 1) : null;
                })
                .filter(Boolean)
        )];
        await fetchAllCategoryData(parsedWSCenID, allIds);

        // --- STEP 6: UIを構築し、シーンを完成させる ---
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        updateInfoPanel();

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error.responseText || error);
        let errorMessage = `Error: ${error.message || 'An unknown error occurred'}.`;
        if (error.responseJSON && error.responseJSON.error) {
            errorMessage = `Error: ${error.responseJSON.error}`;
        }
        if (loaderTextElement) loaderTextElement.textContent = errorMessage;
    } finally {
        if (loaderContainer) loaderContainer.style.display = 'none';
    }
}


// ... (これ以降のヘルパー関数は、ダウンロード関連以外は全て同じです)
// resetScene, parseObjHeader, fetchAllCategoryData, buildAndPopulateCategorizedTree, frameObject,
// createCategoryNode, createObjectNode, handleSelection, applyHighlight, removeAllHighlights,
// zoomToAndIsolate, deIsolateAllObjects, updateInfoPanel, イベントリスナー, animate,
// handleResetView, getVolumeOfSelectedObject, calculateMeshVolume などは全て変更ありません。

function resetScene(){ /* ... 変更なし ... */ }
async function parseObjHeader(objContent){ /* ... 変更なし ... */ }
async function fetchAllCategoryData(wscenId, allElementIds){ /* ... 変更なし ... */ }
async function buildAndPopulateCategorizedTree(){ /* ... 変更なし ... */ }
function frameObject(objectToFrame){ /* ... 変更なし ... */ }
function createCategoryNode(categoryName, objectsInCategory){ /* ... 変更なし ... */ }
function createObjectNode(object, parentULElement, depth){ /* ... 変更なし ... */ }
function handleSelection(target){ /* ... 変更なし ... */ }
const applyHighlight = (target, color) => { /* ... 変更なし ... */ };
const removeAllHighlights = () => { /* ... 変更なし ... */ };
function zoomToAndIsolate(targetObject){ /* ... 変更なし ... */ }
function deIsolateAllObjects(){ /* ... 変更なし ... */ }
function updateInfoPanel(){ /* ... 変更なし ... */ }
viewerContainer.addEventListener('mousedown', (event) => { /* ... 変更なし ... */ });
viewerContainer.addEventListener('mouseup', (event) => { /* ... 変更なし ... */ });
if (closeModelTreeBtn) closeModelTreeBtn.addEventListener('click', () => { /* ... 変更なし ... */ });
if (modelTreeSearch) modelTreeSearch.addEventListener('input', (e) => { /* ... 変更なし ... */ });
if (toggleUiButton) toggleUiButton.addEventListener('click', () => { /* ... 変更なし ... */ });
function onWindowResize(){ /* ... 変更なし ... */ }
window.addEventListener('resize', onWindowResize);
if (modelSelector) modelSelector.addEventListener('change', (event) => { /* ... 変更なし ... */ });
function animate(){ /* ... 変更なし ... */ }
function handleResetView(){ /* ... 変更なし ... */ }
function getVolumeOfSelectedObject(){ /* ... 変更なし ... */ }
function calculateMeshVolume(mesh){ /* ... 変更なし ... */ }
