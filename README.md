@extends('layouts.baselayout')
@section('title', 'CCC - DL/DWH OBJ Viewer')
@section('head')
<meta name="csrf-token" content="{{ csrf_token() }}">
@section('content')
<style>
    /* Main container for the entire application layout */
    #app-container {
        display: flex;
        flex-direction: column;
        height: 87vh;
        width: 100%;
        margin: 0 auto;
    }

    /* Header / Navigation Bar Styles */
    #top-nav {
        flex-shrink: 0;
        height: 50px;
        background-color: #2c3e50;
        color: white;
        display: flex;
        align-items: center;
        justify-content: space-between;
        padding: 0 20px;
        box-sizing: border-box;
        z-index: 50;
    }

    .nav-title {
        font-size: 1.2em;
        font-weight: bold;
        position: absolute;
        right: 50%;
        transform: translateX(-50%);
    }

    .nav-left-group {
        display: flex;
        align-items: center;
        gap: 10px;
    }

    .nav-left-group label {
        font-size: 1em;
        padding: 5px;
    }

    .nav-left-group select {
        padding: 5px;
        border-radius: 4px;
        border: 1px solid #7f8c8d;
        background-color: #34495e;
        color: white;
        font-size: 1em;
    }

    /* Container for the two main columns */
    #main-content {
        display: flex;
        flex-grow: 1;
        overflow-y: scroll;
        height: 86vh;
        width: 100%;
    }

    /* Viewer Container (left Column) */
    #viewer-container {
        flex-grow: 1;
        position: relative;
        overflow: hidden;
        /* The background color of the canvas is now set in JS */
    }

    #viewer-container canvas {
        display: block;
        width: 100%;
        height: 100%;
    }

    /* Model Tree Panel (Left Column) */
    #modelTreePanel {
        /* flex-basis: 20%;
        flex-shrink: 0;
        display: flex;
        flex-direction: column; */
        background-color: #EDF0F2;
        color: #e0e0e0;
        height: 80vh;
        width: 20%;
        overflow-y: scroll;
        position: absolute;
        right: 10px;
        border: 1px solid #666;
    }

    #modelTreePanel .panel-header {
        padding: 10px 15px;
        font-weight: bold;
        border-bottom: 1px solid #fff;
        background-color: #EDF0F2;
        flex-shrink: 0;
        display: flex;
        justify-content: space-between;
        align-items: center;
        color: #000000;
    }

    #modelTreePanel .panel-header input[type="text"] {
        width: calc(100% - 70px);
        box-sizing: border-box;
        padding: 6px 8px;
        background-color: #EDF0F2;
        /* background-color: #aac5de; */
        border: 1px solid #666;
        /* color: #e0e0e0; */
        color: #000000;
        border-radius: 4px;
        margin-left: 10px;
    }

    #modelTreePanel .close-button {
        cursor: pointer;
        font-size: 20px;
        line-height: 1;
    }

    #modelTreeContainer {
        flex-grow: 1;
        overflow-y: auto;
        /* background-color: #fff; */
        /* background-color: #dbeeff !important; */
        background-color: #F5F8FA;
        color: #000;
    }

    #modelTreePanel ul {
        list-style-type: none;
        padding: 0;
        margin: 0;
    }

    #modelTreePanel .tree-item {
        padding: 6px 10px 6px 0;
        border-bottom: 1px solid #fff;
        display: flex;
        align-items: center;
        cursor: pointer;
        font-size: 14px;
    }

    #modelTreePanel .tree-item:hover {
        background-color: #D5E1EE;
    }

    #modelTreePanel .tree-item.selected {
        background-color: #a0c4ff;
        /* Light blue */
        color: #000000;
        /* Black text */
    }

    #modelTreePanel .toggler {
        display: inline-block;
        width: 20px;
        text-align: center;
        cursor: pointer;
        user-select: none;
        margin-right: 5px;
    }

    #modelTreePanel .toggler.empty-toggler {
        color: transparent;
    }

    #modelTreePanel .group-name {
        flex-grow: 1;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
        margin-right: 10px;
    }

    #modelTreePanel .visibility-toggle {
        cursor: pointer;
        font-size: 16px;
        margin-left: auto;
        padding: 0 5px;
    }

    #modelTreePanel .visibility-toggle.visible-icon::before {
        content: '👁️';
    }

    #modelTreePanel .visibility-toggle.hidden-icon::before {
        content: '🚫';
    }

    #modelTreePanel ul ul {
        padding-left: 0;
    }

    /* Info Panel (Positioned inside viewer-container) */
    #objectInfo {
        position: absolute;
        top: 10px;
        left: 10px;
        /* right: 10px; */
        background-color: #EDF0F2;
        color: black;
        padding: 10px;
        border-radius: 5px;
        font-family: monospace;
        white-space: pre-wrap;
        z-index: 10;
    }

    /* Loader (Centered over the viewer-container) */
    #loader-container {
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        display: none;
        /* Hidden by default, shown by JS */
        flex-direction: column;
        justify-content: center;
        align-items: center;
        background-color: rgba(204, 204, 204, 0.7);
        z-index: 1000;
    }

    #loader {
        border: 12px solid #f3f3f3;
        border-radius: 50%;
        border-top: 12px solid #3498db;
        width: 80px;
        height: 80px;
        animation: spin 2s linear infinite;
    }

    #loader-text {
        margin-top: 20px;
        color: #333;
        font-size: 16px;
        font-weight: bold;
    }

    /* 2. 新しく作成したボタン用コンテナのスタイル */
    /* このコンテナをビューワーの下部中央に配置します */
    #viewer-controls-container {
        position: absolute;
        /* 親(viewer-container)を基準に配置 */
        bottom: 25px;
        /* 下から25pxの位置 */
        left: 50%;
        /* 左から50%の位置に移動 */
        transform: translateX(-50%);
        /* コンテナ自身の幅の半分だけ左に戻して中央揃え */
        z-index: 100;
        /* canvasより手前に表示 */

        display: flex;
        /* 中のボタンを横並びにする */
        align-items: center;
        /* ボタンの縦方向の位置を中央に揃える */
        gap: 12px;
        /* ボタンとボタンの間の隙間を12pxに設定 */
    }

    /* 3. ボタンに共通のスタイルを適用 */
    /* UI切り替えボタンのデザインを参考に、両方のボタンに統一感を持たせます */
    #viewer-controls-container button {
        /* background-color: rgba(45, 52, 54, 0.8); */
        background-color: #EDF0F2;
        /* 少し濃いグレー、半透明 */
        color: black;
        font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
        font-weight: bold;
        font-size: 16px;
        border: 1px solid rgba(255, 255, 255, 0.2);
        border-radius: 8px;
        /* 角を丸くする */
        padding: 10px 18px;
        cursor: pointer;
        box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
        /* 影をつけて立体感を出す */
        transition: background-color 0.2s ease, transform 0.1s ease;
        /* アニメーション効果 */
    }

    /* マウスカーソルを合わせた時のスタイル */
    #viewer-controls-container button:hover {
        background-color: rgba(9, 132, 227, 0.9);
        /* ホバー時に色を変更 */
    }

    /* クリックした（押した）時のスタイル */
    #viewer-controls-container button:active {
        transform: translateY(1px);
        /* 少し下に沈むようなエフェクト */
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
    }

    /* ❌ボタンはアイコンなので、少しだけスタイルを微調整 */
    #toggle-ui-button {
        padding: 10px 12px;
        font-size: 20px;
    }

    /* --- スタイルシートに追加 --- */

    #resetViewButton:hover {
        background-color: rgba(60, 60, 60, 0.9);
    }

    #toggle-ui-button:hover {
        background-color: #34495e;
    }

    @keyframes spin {
        0% {
            transform: rotate(0deg);
        }

        100% {
            transform: rotate(360deg);
        }
    }
</style>
</head>
<input type="hidden" id="hidLoginID" name="hidLoginID" value="{{Session::get('login_user_id')}}" />
@if (Session::has('access_token'))
<input type="hidden" id="access_token" value="{{Session::get('access_token')}}" />
@endif

<body>
    <div id="app-container">
        <!-- Top Navigation Bar -->
        <div id="top-nav">
            <!-- This empty div acts as a spacer for the left side -->
            <div class="nav-left-group">
                <label for="model-selector">Select Model:</label>
                <select id="model-selector">
                    <option value="GF本社移転">GF本社移転</option>
                    <option value="BIKEN15号棟">BIKEN15号棟</option>
                    <option value="パナソニックエナジー西門真地区">パナソニックエナジー西門真地区</option>
                    <!-- Add more models here -->
                </select>
            </div>
            <div></div>
            <div class="nav-title">3D Viewer</div>
        </div>

        <!-- Main Content Area with two columns -->
        <div id="main-content">
            <!-- Left Column: 3D Viewer -->
            <div id="viewer-container">
                <!-- The Three.js canvas will be appended here by the script -->
                <div id="loader-container">
                    <div id="loader"></div>
                    <div id="loader-text">Loading 3D Model...</div>
                </div>
                <div id="objectInfo">None</div>
                <!-- Add the new toggle button outside the main layout container -->
                <div id="viewer-controls-container">
                    <!-- 「全体表示」ボタン -->
                    <button id="resetViewButton" title="全体を表示（Reset View）">選択解除</button>

                    <!-- 「UI表示/非表示」ボタン -->
                    <button id="toggle-ui-button" title="UIパネルの表示/非表示">❌</button>
                </div>
            </div>
            <!-- Right Column: Model Tree -->
            <div id="modelTreePanel">
                <div class="panel-header">
                    <span>モデル</span>
                </div>
                <div id="modelTreeContainer">
                    <ul id="modelTreeList"></ul>
                </div>
            </div>
        </div>
    </div>
    <!-- <script type="module" src="{{ asset('/js/objViewerStandard.js') }}"></script> -->

    <script type="module" src="{{ asset('/js/objViewer.js') }}"></script>
    @endsection

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
const viewerContainer = document.getElementById('viewer-container'); // NEW: Get the right-side container
const toggleUiButton = document.getElementById('toggle-ui-button'); // NEW
// --- NEW: リセットボタンの要素を取得 ---
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
const highlightColorSingle = new THREE.Color(0xa0c4ff); // NEW: Light Blue (matches CSS)
const elementIdDataMap = new Map(); // Stores all fetched element data


// --- Model Configuration ---
// Create an object to store the file paths for each model
const models = {
    'GF本社移転': {
        obj: '240324_GF本社移転_2022_20250627.obj',
        mtl: '240324_GF本社移転_2022_20250627.mtl',
        // You can add other model-specific settings here if needed
    },
    'BIKEN15号棟': {
        obj: '240627_BIKEN15号棟_2022_20250630.obj', // Placeholder filename
        mtl: '240627_BIKEN15号棟_2022_20250630.mtl', // Placeholder filename
    },
    'パナソニックエナジー西門真地区': {
        obj: '240628_パナソニックエナジー西門真地区_2022__20250630.obj', // Placeholder filename
        mtl: '240628_パナソニックエナジー西門真地区_2022__20250630.mtl', // Placeholder filename
    },
    // Add other models here
};


// --- Scene Setup, Camera, Renderer, Lighting, Controls ---
const scene = new THREE.Scene();

// --- Autodesk-Style Gradient Background ---
// This creates a plane that always fills the camera's view.
const backgroundGeometry = new THREE.PlaneGeometry(2, 2, 1, 1);
const backgroundMaterial = new THREE.ShaderMaterial({
    // This GLSL code runs on the GPU
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = vec4(position.xy, 1.0, 1.0);
        }
    `,
    fragmentShader: `
        uniform vec3 topColor;
        uniform vec3 bottomColor;
        varying vec2 vUv;
        void main() {
            gl_FragColor = vec4(mix(bottomColor, topColor, vUv.y), 1.0);
        }
    `,
    uniforms: {
        // Define the colors for the gradient
        topColor: { value: new THREE.Color(0xd8e3ee) }, // Lighter sky blue
        bottomColor: { value: new THREE.Color(0xf0f0f0) } // Light gray ground
    },
    depthWrite: false // Don't interfere with the 3D objects
});

const backgroundMesh = new THREE.Mesh(backgroundGeometry, backgroundMaterial);
// Tell Three.js not to treat this mesh like a regular 3D object
backgroundMesh.renderOrder = -1; // Render it first (in the background)
scene.add(backgroundMesh);

//カメラの設定
const camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 0.1, 20000);
camera.position.copy(initialCameraPosition);
camera.lookAt(initialCameraLookAt);

//レンダラーの設定
const renderer = new THREE.WebGLRenderer({ antialias: true });
// renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
viewerContainer.appendChild(renderer.domElement); // <-- Append canvas to the new container
// document.body.appendChild(renderer.domElement);

// --- 変数定義 ---
// マウスを押した位置を記憶するための変数
let mouseDownPosition = new THREE.Vector2();


//ライトの設定
const ambientLight = new THREE.AmbientLight(0x606060, 2);
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xFFFFFF, 2.5);
directionalLight.position.set(50, 100, 75);
directionalLight.castShadow = true;
directionalLight.shadow.mapSize.width = 2048;
directionalLight.shadow.mapSize.height = 2048;
scene.add(directionalLight);

const hemiLight = new THREE.HemisphereLight(0xffffff, 0x8d8d8d, 1.5);
hemiLight.position.set(0, 50, 0);
scene.add(hemiLight);

//コントロール操作
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;
// --- Helper Functions ---

/* ajax通信トークン定義 */
var CSRF_TOKEN = $('meta[name="csrf-token"]').attr('content');

$(document).ready(function () {
    var login_user_id = $("#hidLoginID").val();
    var img_src = "/DL_DWH.png";
    var url = "DLDWH/objviewer";
    var content_name = "OBJビューア";
    recordAccessHistory(login_user_id, img_src, url, content_name);
    // --- Start Application ---
    onWindowResize();
    loadModel('GF本社移転'); // 'GF' is the default model
    animate();

});


// --- REFACTORED Main Application Flow into a Reusable Function ---
async function loadModel(modelKey) {
    // 0. Reset the scene and state
    if (loadedObjectModelRoot) {
        scene.remove(loadedObjectModelRoot);
    }
    // Clear all previous data
    loadedObjectModelRoot = null;
    selectedObjectOrGroup = null;
    originalMeshMaterials.clear();
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
    elementIdDataMap.clear();
    modelTreeList.innerHTML = '';
    if (modelTreePanel) modelTreePanel.style.display = 'none';

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';

        const modelFiles = models[modelKey];
        if (!modelFiles) {
            throw new Error(`Model key "${modelKey}" not found in configuration.`);
        }

        const assetPath = '/ccc/public/objFiles/';
        const objFileName = modelFiles.obj;
        const mtlFileName = modelFiles.mtl;
        const fullObjPath = assetPath + objFileName;

        // Step 1: Parse header info
        await parseObjHeader(fullObjPath);

        // Step 2: Load Materials
        if (loaderTextElement) loaderTextElement.textContent = `Loading Materials...`;
        const mtlLoader = new MTLLoader();
        mtlLoader.setPath(assetPath);
        const materialsCreator = await new Promise((resolve, reject) => {
            mtlLoader.load(mtlFileName, resolve, undefined, reject);
        });
        materialsCreator.preload();

        // Step 3: Load Geometry
        const objLoader = new OBJLoader();
        objLoader.setMaterials(materialsCreator);
        const object = await new Promise((resolve, reject) => {
            objLoader.load(fullObjPath, resolve, (xhr) => {
                if (loaderTextElement) {
                    const percent = Math.round(xhr.loaded / xhr.total * 100);
                    loaderTextElement.textContent = isFinite(percent) && percent < 100 ?
                        `Loading 3D Geometry: ${percent}%` : `Processing Geometry...`;
                }
            }, reject);
        });

        loadedObjectModelRoot = object;

        // Step 4: Process Model (center, scale, rotate)
        const initialBox = new THREE.Box3().setFromObject(object);
        const initialCenter = initialBox.getCenter(new THREE.Vector3());
        object.traverse((child) => {
            if (child.isMesh) {
                child.geometry.translate(-initialCenter.x, -initialCenter.y, -initialCenter.z);
                child.castShadow = true;
                child.receiveShadow = true;
            }
        });
        object.position.set(0, 0, 0);
        const scaledBox = new THREE.Box3().setFromObject(object);
        const maxDim = Math.max(scaledBox.getSize(new THREE.Vector3()).x, scaledBox.getSize(new THREE.Vector3()).y, scaledBox.getSize(new THREE.Vector3()).z);
        const desiredMaxDimension = 150;
        if (maxDim > 0) {
            const scale = desiredMaxDimension / maxDim;
            object.scale.set(scale, scale, scale);
        }
        object.rotation.x = -Math.PI / 2;
        // object.rotation.y = -Math.PI / 2;

        // Step 5: Fetch category data
        const allIds = [];
        loadedObjectModelRoot.traverse(child => {
            if (child.name && child.parent === loadedObjectModelRoot) {
                const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                if (splitIndex > 0) allIds.push(child.name.substring(splitIndex + 1));
            }
        });
        await fetchAllCategoryData(parsedWSCenID, [...new Set(allIds)]);

        // Step 6: Build the tree UI
        buildAndPopulateCategorizedTree();

        // Step 7: Add model to scene, frame it, and hide loader
        scene.add(object);
        frameObject(object);

        if (loaderContainer) loaderContainer.style.display = 'none';

        updateInfoPanel(); // Clear the info panel

    } catch (error) {
        console.error('Failed to initialize the viewer:', error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model: ${modelKey}.`;
    }
}

async function parseObjHeader(filePath) {
    try {
        const response = await fetch(filePath);
        if (!response.ok) {
            console.error(`Failed to fetch OBJ for header parsing: ${response.statusText}`);
            return;
        }
        const text = await response.text();
        const lines = text.split(/\r?\n/);
        if (lines.length > 0) {
            const firstLine = lines[0].trim();
            if (firstLine.startsWith("# ")) {
                const content = firstLine.substring(2).trim();
                const pattern1Match = content.match(/^([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
                if (pattern1Match) {
                    parsedWSCenID = pattern1Match[1];
                    parsedPJNo = pattern1Match[2];
                    return;
                }
                if (content.includes("ワークシェアリングされてない") && (content.includes("_PJNo無し") || content.includes("＿PJNo無し"))) {
                    parsedWSCenID = "";
                    parsedPJNo = "";
                    return;
                }
            }
        }
    } catch (error) {
        console.error("Error fetching or parsing OBJ header:", error);
    }
}

async function fetchAllCategoryData(wscenId, allElementIds) {
    if (allElementIds.length === 0) {
        console.log("No element IDs to fetch data for.");
        return Promise.resolve(); // Immediately resolve if there's nothing to do
    }
    const batchSize = 900;
    for (let i = 0; i < allElementIds.length; i += batchSize) {
        const batch = allElementIds.slice(i, i + batchSize);
        if (loaderTextElement) loaderTextElement.textContent = `Fetching Categories... (${i + batch.length}/${allElementIds.length})`;
        await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/DLDWH/getDatas",
                data: { _token: CSRF_TOKEN, WSCenID: wscenId, ElementIds: batch },
                success: function (data) {
                    for (const elementId in data) {
                        elementIdDataMap.set(elementId, data[elementId]);
                    }
                    resolve();
                },
                error: function (err) {
                    console.error(`Error fetching batch starting at index ${i}:`, err);
                    reject(err);
                }
            });
        });
    }
}

function buildAndPopulateCategorizedTree() {
    if (!loadedObjectModelRoot || !modelTreeList) return;
    if (loaderTextElement) loaderTextElement.textContent = "Building model tree...";
    const categorizedObjects = {};
    loadedObjectModelRoot.traverse(child => {
        if ((child.isGroup || child.isMesh) && child.name && child.parent === loadedObjectModelRoot) {
            let rawName = child.name;
            let displayId = null;
            const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
            if (splitIndex > 0) displayId = rawName.substring(splitIndex + 1);
            let category = "カテゴリー無し";
            if (displayId && elementIdDataMap.has(displayId)) {
                category = elementIdDataMap.get(displayId)['カテゴリー名'] || "カテゴリー無し";
            } else if (!displayId) {
                category = "名称未分類";
            }
            if (!categorizedObjects[category]) categorizedObjects[category] = [];
            categorizedObjects[category].push(child);
        }
    });
    modelTreeList.innerHTML = '';
    Object.keys(categorizedObjects).sort().forEach(categoryName => {
        createCategoryNode(categoryName, categorizedObjects[categoryName]);
    });
    if (modelTreePanel) modelTreePanel.style.display = 'block';
}

function frameObject(objectToFrame) {
    const box = new THREE.Box3().setFromObject(objectToFrame);
    if (box.isEmpty()) {
        camera.position.set(50, 50, 50);
        camera.lookAt(0, 0, 0);
        controls.target.set(0, 0, 0);
        controls.update();
        return;
    }
    const center = box.getCenter(new THREE.Vector3());
    const sphere = box.getBoundingSphere(new THREE.Sphere());
    const radius = sphere.radius;
    initialCameraLookAt.copy(center);
    controls.target.copy(center);
    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    const distance = radius / Math.sin(fovInRadians / 2);
    const zoomOutFactor = 1.3;
    const finalDistance = distance * zoomOutFactor;
    const cameraDirection = new THREE.Vector3(1, 0.6, 1).normalize();
    initialCameraPosition.copy(center).addScaledVector(cameraDirection, finalDistance);
    camera.position.copy(initialCameraPosition);
    camera.lookAt(initialCameraLookAt);
    controls.update();
}

function createCategoryNode(categoryName, objectsInCategory) {
    const categoryLi = document.createElement('li');
    const itemContent = document.createElement('div');
    itemContent.className = 'tree-item';
    itemContent.style.fontWeight = 'bold';
    const toggler = document.createElement('span');
    toggler.className = 'toggler';
    toggler.textContent = '▼';
    itemContent.appendChild(toggler);
    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = `${categoryName} (${objectsInCategory.length})`;
    itemContent.appendChild(nameSpan);
    const subList = document.createElement('ul');
    toggler.addEventListener('click', (e) => {
        e.stopPropagation();
        const isCollapsed = subList.style.display === 'none';
        subList.style.display = isCollapsed ? 'block' : 'none';
        toggler.textContent = isCollapsed ? '▼' : '▶';
    });
    categoryLi.appendChild(itemContent);
    categoryLi.appendChild(subList);
    modelTreeList.appendChild(categoryLi);
    objectsInCategory.forEach(object => createObjectNode(object, subList, 1));
}

function createObjectNode(object, parentULElement, depth) {
    const listItem = document.createElement('li');
    listItem.dataset.uuid = object.uuid;
    const itemContent = document.createElement('div');
    itemContent.className = 'tree-item';
    itemContent.style.paddingLeft = `${depth * 15 + 10}px`;
    const toggler = document.createElement('span');
    toggler.className = 'toggler empty-toggler';
    toggler.innerHTML = ' ';
    itemContent.appendChild(toggler);
    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = object.name;
    nameSpan.title = object.name;
    itemContent.appendChild(nameSpan);
    const visibilityToggle = document.createElement('span');
    visibilityToggle.className = 'visibility-toggle visible-icon';
    visibilityToggle.title = 'Hide';
    itemContent.appendChild(visibilityToggle);
    listItem.appendChild(itemContent);
    parentULElement.appendChild(listItem);
    itemContent.addEventListener('click', () => handleSelection(object));
    visibilityToggle.addEventListener('click', (e) => {
        e.stopPropagation();
        object.visible = !object.visible;
        visibilityToggle.classList.toggle('visible-icon', object.visible);
        visibilityToggle.classList.toggle('hidden-icon', !object.visible);
        visibilityToggle.title = object.visible ? 'Hide' : 'Show';
        if (!object.visible && selectedObjectOrGroup && selectedObjectOrGroup.uuid === object.uuid) {
            handleSelection(null);
        }
    });
}


function handleSelection(target) {
    removeAllHighlights();
    deIsolateAllObjects();
    let newSelection = null;
    if (target && (!selectedObjectOrGroup || selectedObjectOrGroup.uuid !== target.uuid)) {
        newSelection = target;
    }
    selectedObjectOrGroup = newSelection;
    if (selectedObjectOrGroup) {
        applyHighlight(selectedObjectOrGroup, highlightColorSingle);
        zoomToAndIsolate(selectedObjectOrGroup);
    }
    document.querySelectorAll('#modelTreePanel .tree-item.selected').forEach(el => el.classList.remove('selected'));
    if (selectedObjectOrGroup) {
        const treeItemDiv = document.querySelector(`#modelTreePanel li[data-uuid="${selectedObjectOrGroup.uuid}"] .tree-item`);
        if (treeItemDiv) treeItemDiv.classList.add('selected');
    }
    updateInfoPanel();
}

const applyHighlight = (target, color) => {
    if (!target) return;
    const meshesToHighlight = [];
    if (target.isMesh) meshesToHighlight.push(target);
    else if (target.isGroup) target.traverse(child => { if (child.isMesh) meshesToHighlight.push(child); });
    meshesToHighlight.forEach(mesh => {
        if (mesh.material) {
            if (!originalMeshMaterials.has(mesh.uuid)) originalMeshMaterials.set(mesh.uuid, mesh.material);
            if (Array.isArray(mesh.material)) {
                mesh.material = mesh.material.map(originalMat => {
                    const highlightMat = originalMat.clone();
                    if (highlightMat.color) highlightMat.color.set(color);
                    else if (highlightMat.emissive) highlightMat.emissive.set(color);
                    return highlightMat;
                });
            } else {
                const highlightMat = mesh.material.clone();
                if (highlightMat.color) highlightMat.color.set(color);
                else if (highlightMat.emissive) highlightMat.emissive.set(color);
                mesh.material = highlightMat;
            }
        }
    });
};

const removeAllHighlights = () => {
    originalMeshMaterials.forEach((originalMaterialOrArray, meshUuid) => {
        const mesh = scene.getObjectByProperty('uuid', meshUuid);
        if (mesh && mesh.isMesh) mesh.material = originalMaterialOrArray;
    });
    originalMeshMaterials.clear();
};

function zoomToAndIsolate(targetObject) {
    if (!targetObject) return;
    deIsolateAllObjects();
    isIsolateModeActive = true;
    const box = new THREE.Box3().setFromObject(targetObject);
    if (box.isEmpty()) { isIsolateModeActive = false; return; }
    const center = box.getCenter(new THREE.Vector3());
    const sphere = box.getBoundingSphere(new THREE.Sphere());
    const radius = sphere.radius;
    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    let distance = radius / Math.sin(fovInRadians / 2);
    distance = distance * 1.5;
    const offsetDirection = camera.position.clone().sub(controls.target).normalize();
    if (offsetDirection.lengthSq() === 0) offsetDirection.set(0.5, 0.5, 1).normalize();
    camera.position.copy(center).addScaledVector(offsetDirection, distance);
    controls.target.copy(center);
    controls.update();
    loadedObjectModelRoot.traverse((object) => {
        if (object.isMesh) {
            let isPartOfSelectedTarget = false;
            let temp = object;
            while (temp) {
                if (temp === targetObject) { isPartOfSelectedTarget = true; break; }
                temp = temp.parent;
            }
            if (!isPartOfSelectedTarget) {
                if (!originalObjectPropertiesForIsolate.has(object.uuid)) {
                    originalObjectPropertiesForIsolate.set(object.uuid, { material: object.material, visible: object.visible });
                }
                if (object.visible) {
                    const materials = Array.isArray(object.material) ? object.material : [object.material];
                    const newMaterials = materials.map(mat => {
                        const newMat = mat.clone();
                        newMat.transparent = true;
                        newMat.opacity = 0.1;
                        return newMat;
                    });
                    object.material = Array.isArray(object.material) ? newMaterials : newMaterials[0];
                }
            } else {
                if (originalObjectPropertiesForIsolate.has(object.uuid)) {
                    const props = originalObjectPropertiesForIsolate.get(object.uuid);
                    object.material = props.material;
                    object.visible = props.visible;
                    originalObjectPropertiesForIsolate.delete(object.uuid);
                } else {
                    const materials = Array.isArray(object.material) ? object.material : [object.material];
                    materials.forEach(mat => { mat.transparent = false; mat.opacity = 1.0; });
                    object.visible = true;
                }
            }
        }
    });
}

function deIsolateAllObjects() {
    if (!isIsolateModeActive) return;
    originalObjectPropertiesForIsolate.forEach((props, uuid) => {
        const object = scene.getObjectByProperty('uuid', uuid);
        if (object && object.isMesh) {
            object.material = props.material;
            object.visible = props.visible;
        }
    });
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
}

function updateInfoPanel() {
    // if (!objectInfoPanel) return;
    let headerInfo = `WSCenID: ${parsedWSCenID || "N/A"}\nPJNo: ${parsedPJNo || "N/A"}\n----\n`;
    if (parsedWSCenID === "" && parsedPJNo === "") headerInfo = `WSCenID: \nPJNo: \n----\n`;
    if (selectedObjectOrGroup) {
        let rawName = selectedObjectOrGroup.name || "Unnamed";
        let displayName = rawName;
        let displayId = "N/A";
        const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
        if (splitIndex > 0) {
            displayName = rawName.substring(0, splitIndex);
            displayId = rawName.substring(splitIndex + 1);
        } else displayName = rawName;
        let selectionInfo = `名前: ${displayName}\nID: ${displayId}\n`;
        if (displayId && elementIdDataMap.has(displayId)) {
            const data = elementIdDataMap.get(displayId);
            selectionInfo += `\nカテゴリー名: ${data['カテゴリー名'] || "N/A"}\nファミリ名: ${data['ファミリ名'] || "N/A"}\nタイプ_ID: ${data['タイプ_ID'] || "N/A"}`;
        } else {
            selectionInfo += '\nカテゴリー名: "" \nファミリ名: ""';
        }
        objectInfoPanel.textContent = headerInfo + selectionInfo;
    } else {
        objectInfoPanel.textContent = headerInfo + 'None Selected';
    }
}

// マウスのボタンが押された時の処理
viewerContainer.addEventListener('mousedown', (event) => {
    mouseDownPosition.set(event.clientX, event.clientY);
}, false);

// マウスのボタンが離された時の処理（クリックかドラッグかを判定）
viewerContainer.addEventListener('mouseup', (event) => {
    const mouseUpPosition = new THREE.Vector2(event.clientX, event.clientY);
    const distance = mouseDownPosition.distanceTo(mouseUpPosition);

    const DRAG_THRESHOLD = 5;
    if (distance > DRAG_THRESHOLD) {
        return; // ドラッグ操作だったので、選択処理はしない
    }

    // 以下はクリックだった場合の選択処理
    if (!loadedObjectModelRoot) return;
    const rect = viewerContainer.getBoundingClientRect();
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);
    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let current = intersects[0].object;
        while (current && current.parent !== loadedObjectModelRoot && current !== loadedObjectModelRoot && current.parent !== scene) {
            current = current.parent;
        }
        if (current) newlyClickedTarget = current;
    }
    handleSelection(newlyClickedTarget);
}, false);


if (closeModelTreeBtn) closeModelTreeBtn.addEventListener('click', () => { if (modelTreePanel) modelTreePanel.style.display = 'none'; });

if (modelTreeSearch) {
    modelTreeSearch.addEventListener('input', (e) => {
        const searchTerm = e.target.value.toLowerCase().trim();
        const allListItems = modelTreeList.querySelectorAll('li');
        allListItems.forEach(li => {
            const itemContentDiv = li.querySelector('.tree-item');
            if (!itemContentDiv) return;
            const nameSpan = itemContentDiv.querySelector('.group-name');
            if (!nameSpan) return;
            const itemName = nameSpan.textContent.toLowerCase();
            const isMatch = itemName.includes(searchTerm);

            if (isMatch) {
                li.style.display = '';
                let parentLi = li.parentElement.closest('li');
                while (parentLi) {
                    parentLi.style.display = '';
                    const parentSubList = parentLi.querySelector('ul');
                    if (parentSubList) parentSubList.style.display = 'block';
                    const parentToggler = parentLi.querySelector('.toggler:not(.empty-toggler)');
                    if (parentToggler) parentToggler.textContent = '▼';
                    parentLi = parentLi.parentElement.closest('li');
                }
            } else {
                li.style.display = 'none';
            }
        });
        if (searchTerm === "") {
            allListItems.forEach(li => li.style.display = '');
        }
    });
}

// --- NEW: Event listener for the UI Toggle Button (Request ③) ---
if (toggleUiButton) {
    toggleUiButton.addEventListener('click', () => {
        // Check the current visibility of one of the panels to decide the action
        const isVisible = modelTreePanel.style.display !== 'none';

        if (isVisible) {
            // Hide panels
            if (modelTreePanel) modelTreePanel.style.display = 'none';
            // if (objectInfoPanel) objectInfoPanel.style.display = 'none';
            toggleUiButton.textContent = '📊'; // Change icon to "show"
            toggleUiButton.title = "Show UI Panels";
        } else {
            // Show panels
            if (modelTreePanel) modelTreePanel.style.display = 'block';
            // if (objectInfoPanel) objectInfoPanel.style.display = 'block';
            toggleUiButton.textContent = '❌'; // Change icon to "hide"
            toggleUiButton.title = "Hide UI Panels";
        }
    });
}

// This function now correctly resizes the renderer based on its container's dimensions
function onWindowResize() {
    const { clientWidth, clientHeight } = viewerContainer;

    camera.aspect = clientWidth / clientHeight;
    camera.updateProjectionMatrix();

    renderer.setSize(clientWidth, clientHeight);
}
window.addEventListener('resize', onWindowResize);

// --- NEW: Event listener for the model selector dropdown ---
if (modelSelector) {
    modelSelector.addEventListener('change', (event) => {
        const selectedModelKey = event.target.value;
        loadModel(selectedModelKey);
    });
}


function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.autoClearColor = false;
    renderer.render(scene, camera);
}



    
