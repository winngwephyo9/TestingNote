
<img width="1908" height="818" alt="image" src="https://github.com/user-attachments/assets/29c03e80-e185-4cb5-b381-53dcd51dab92" />

/**
 * Frames a target object in the camera's view.
 * This version includes an immediate controls.update() call to ensure
 * the change is not lost during the animation loop, fixing the multi-click reset issue.
 *
 * @param {THREE.Object3D} objectToFrame - The object or group to frame.
 */
function frameObject(objectToFrame) {
    const box = new THREE.Box3().setFromObject(objectToFrame);
    if (box.isEmpty()) {
        // Fallback for empty objects to prevent errors
        camera.position.set(50, 50, 50);
        controls.target.set(0, 0, 0);
        controls.update(); // Update controls even on fallback
        return;
    }

    const center = box.getCenter(new THREE.Vector3());
    const sphere = box.getBoundingSphere(new THREE.Sphere());

    // Use a minimum radius to prevent issues with very small or flat objects
    const radius = Math.max(sphere.radius, 1);

    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    const distance = (radius / Math.sin(fovInRadians / 2)) * 1.3; // 1.3 zoomOutFactor

    const cameraDirection = new THREE.Vector3(1, 0.6, 1).normalize();
    const newPosition = center.clone().addScaledVector(cameraDirection, distance);

    // --- THE CRITICAL FIX ---

    // 1. Set the new camera position and the point to look at.
    camera.position.copy(newPosition);
    controls.target.copy(center);

    // 2. Force the controls to immediately synchronize with the new state.
    // This is the key change. It tells OrbitControls to cancel any ongoing
    // animation and adopt the new position and target as its current state.
    // The damping will now apply to user input *from this new position*.
    controls.update();
}

<img width="1901" height="817" alt="image" src="https://github.com/user-attachments/assets/81099a3f-c56a-4ac5-bcd0-b9c1b54f9c41" />


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

// --- Configuration ---
const CSRF_TOKEN = document.querySelector('meta[name="csrf-token"]').getAttribute('content');
const ASSET_PATH = '/ccc/public/objFiles/';
const API_URL = url_prefix + "/DLDWH/getDatas";
const DRAG_THRESHOLD = 5;
const HIGHLIGHT_COLOR_SINGLE = new THREE.Color(0xa0c4ff); // Light Blue

const MODELS = {
    'GF本社移転': {
        obj: '240324_GF本社移転_2022_20250627.obj',
        mtl: '240324_GF本社移転_2022_20250627.mtl',
    },
    'BIKEN15号棟': {
        obj: '240627_BIKEN15号棟_2022_20250630.obj',
        mtl: '240627_BIKEN15号棟_2022_20250630.mtl',
    },
    'パナソニックエナジー西門真地区': {
        obj: '240628_パナソニックエナジー西門真地区_2022__20250630.obj',
        mtl: '240628_パナソニックエナジー西門真地区_2022__20250630.mtl',
    },
};

// --- UI Elements ---
const ui = {
    loaderContainer: document.getElementById('loader-container'),
    loaderText: document.getElementById('loader-text'),
    modelTreePanel: document.getElementById('modelTreePanel'),
    modelTreeList: document.getElementById('modelTreeList'),
    objectInfoPanel: document.getElementById('objectInfo'),
    modelSelector: document.getElementById('model-selector'),
    viewerContainer: document.getElementById('viewer-container'),
    toggleUiButton: document.getElementById('toggle-ui-button'),
    resetViewButton: document.getElementById('resetViewButton'),
};

// --- Application State ---
let state = {
    loadedObjectModelRoot: null,
    selectedObjectOrGroup: null,
    isIsolateModeActive: false,
    parsedWSCenID: "",
    parsedPJNo: "",
    initialCameraPosition: new THREE.Vector3(10, 10, 10),
    initialCameraLookAt: new THREE.Vector3(0, 0, 0),
    originalMeshMaterials: new Map(),
    originalObjectPropertiesForIsolate: new Map(),
    elementIdDataMap: new Map(),
    mouseDownPosition: new THREE.Vector2(),
};

// --- Core Three.js Components ---
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 0.1, 20000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
const controls = new OrbitControls(camera, renderer.domElement);
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

/**
 * Initializes the entire application.
 */
function init() {
    setupScene();
    setupControls();
    setupEventListeners();
    recordAccessHistoryIfNeeded();

    loadModel(ui.modelSelector.value); // Load the default selected model
    animate();
}

/**
 * Sets up the 3D scene, including background, lighting, and renderer.
 */
function setupScene() {
    // Gradient Background
    const backgroundGeometry = new THREE.PlaneGeometry(2, 2, 1, 1);
    const backgroundMaterial = new THREE.ShaderMaterial({
        vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = vec4(position.xy, 1.0, 1.0); }`,
        fragmentShader: `uniform vec3 topColor; uniform vec3 bottomColor; varying vec2 vUv; void main() { gl_FragColor = vec4(mix(bottomColor, topColor, vUv.y), 1.0); }`,
        uniforms: {
            topColor: { value: new THREE.Color(0xd8e3ee) },
            bottomColor: { value: new THREE.Color(0xf0f0f0) }
        },
        depthWrite: false
    });
    const backgroundMesh = new THREE.Mesh(backgroundGeometry, backgroundMaterial);
    backgroundMesh.renderOrder = -1;
    scene.add(backgroundMesh);

    // Camera
    camera.position.copy(state.initialCameraPosition);
    camera.lookAt(state.initialCameraLookAt);

    // Renderer
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    ui.viewerContainer.appendChild(renderer.domElement);

    // Lighting
    scene.add(new THREE.AmbientLight(0x606060, 2));
    const directionalLight = new THREE.DirectionalLight(0xFFFFFF, 2.5);
    directionalLight.position.set(50, 100, 75);
    directionalLight.castShadow = true;
    directionalLight.shadow.mapSize.width = 2048;
    directionalLight.shadow.mapSize.height = 2048;
    scene.add(directionalLight);
    scene.add(new THREE.HemisphereLight(0xffffff, 0x8d8d8d, 1.5));
}

/**
 * Configures the OrbitControls.
 */
function setupControls() {
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
}

/**
 * Attaches all necessary event listeners.
 */
function setupEventListeners() {
    window.addEventListener('resize', onWindowResize);
    ui.modelSelector.addEventListener('change', (e) => loadModel(e.target.value));
    ui.resetViewButton.addEventListener('click', handleResetView);
    ui.toggleUiButton.addEventListener('click', handleToggleUI);
    ui.viewerContainer.addEventListener('mousedown', onMouseDown);
    ui.viewerContainer.addEventListener('mouseup', onMouseUp);
}


/**
 * Main function to load a new 3D model into the scene.
 * @param {string} modelKey - The key for the model in the MODELS configuration.
 */
async function loadModel(modelKey) {
    resetStateAndScene();
    showLoader(true, 'Loading 3D Model...');

    try {
        const modelFiles = MODELS[modelKey];
        if (!modelFiles) throw new Error(`Model key "${modelKey}" not found.`);

        const fullObjPath = ASSET_PATH + modelFiles.obj;

        await parseObjHeader(fullObjPath);

        showLoader(true, 'Loading Materials...');
        const mtlLoader = new MTLLoader().setPath(ASSET_PATH);
        const materialsCreator = await mtlLoader.loadAsync(modelFiles.mtl);
        materialsCreator.preload();

        const objLoader = new OBJLoader().setMaterials(materialsCreator);
        const object = await objLoader.loadAsync(fullObjPath, (xhr) => {
            const percent = Math.round(xhr.loaded / xhr.total * 100);
            const message = isFinite(percent) && percent < 100 ? `Loading 3D Geometry: ${percent}%` : `Processing Geometry...`;
            showLoader(true, message);
        });

        state.loadedObjectModelRoot = object;
        processLoadedModel(object);

        const allElementIds = getAllElementIdsFromModel(object);
        await fetchAllCategoryData(state.parsedWSCenID, allElementIds);

        buildAndPopulateCategorizedTree();

        scene.add(object);
        frameObject(object);
        updateInfoPanel();

    } catch (error) {
        console.error(`Failed to initialize viewer for model "${modelKey}":`, error);
        showLoader(true, `Error loading model: ${modelKey}.`);
    } finally {
        showLoader(false);
    }
}

/**
 * Resets the application state and clears the 3D scene for a new model.
 */
function resetStateAndScene() {
    if (state.loadedObjectModelRoot) {
        scene.remove(state.loadedObjectModelRoot);
    }
    state.loadedObjectModelRoot = null;
    state.selectedObjectOrGroup = null;
    state.isIsolateModeActive = false;
    state.parsedWSCenID = "";
    state.parsedPJNo = "";
    state.originalMeshMaterials.clear();
    state.originalObjectPropertiesForIsolate.clear();
    state.elementIdDataMap.clear();
    ui.modelTreeList.innerHTML = '';
    ui.modelTreePanel.style.display = 'none';
}

/**
 * Processes the loaded 3D model: centers, scales, and rotates it.
 * @param {THREE.Object3D} object - The loaded model group.
 */
function processLoadedModel(object) {
    const box = new THREE.Box3().setFromObject(object);
    const center = box.getCenter(new THREE.Vector3());

    object.traverse((child) => {
        if (child.isMesh) {
            child.geometry.translate(-center.x, -center.y, -center.z);
            child.castShadow = true;
            child.receiveShadow = true;
        }
    });

    object.position.set(0, 0, 0);

    const scaledBox = new THREE.Box3().setFromObject(object);
    const maxDim = Math.max(...scaledBox.getSize(new THREE.Vector3()).toArray());
    const desiredMaxDimension = 150;
    if (maxDim > 0) {
        const scale = desiredMaxDimension / maxDim;
        object.scale.set(scale, scale, scale);
    }

    object.rotation.x = -Math.PI / 2;
}


// --- Event Handlers ---

/**
 * Handles the click event on the Reset View button.
 */
function handleResetView() {
    // Clear selection, which also handles de-isolation and highlighting.
    handleSelection(null);
    // Frame the entire model.
    if (state.loadedObjectModelRoot) {
        frameObject(state.loadedObjectModelRoot);
    }
}


/**
 * Handles toggling the visibility of UI panels.
 */
function handleToggleUI() {
    const isVisible = ui.modelTreePanel.style.display !== 'none';
    ui.modelTreePanel.style.display = isVisible ? 'none' : 'block';
    // ui.objectInfoPanel.style.display = isVisible ? 'none' : 'block';
    ui.toggleUiButton.textContent = isVisible ? '📊' : '❌';
    ui.toggleUiButton.title = isVisible ? 'Show UI Panels' : 'Hide UI Panels';
}

/**
 * Handles window resize events to keep the viewer correctly sized.
 */
function onWindowResize() {
    const { clientWidth, clientHeight } = ui.viewerContainer;
    camera.aspect = clientWidth / clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(clientWidth, clientHeight);
}

/**
 * Records mouse position on mousedown.
 */
function onMouseDown(event) {
    state.mouseDownPosition.set(event.clientX, event.clientY);
}

/**
 * Determines if a click or drag occurred on mouseup and handles selection.
 */
function onMouseUp(event) {
    const mouseUpPosition = new THREE.Vector2(event.clientX, event.clientY);
    if (state.mouseDownPosition.distanceTo(mouseUpPosition) > DRAG_THRESHOLD) {
        return; // It was a drag, not a click.
    }

    if (!state.loadedObjectModelRoot) return;

    const rect = ui.viewerContainer.getBoundingClientRect();
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(state.loadedObjectModelRoot.children, true);

    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let current = intersects[0].object;
        // Traverse up to find the main group parent under the root model object
        while (current && current.parent !== state.loadedObjectModelRoot && current.parent !== scene) {
            current = current.parent;
        }
        if (current) {
            newlyClickedTarget = current;
        }
    }
    handleSelection(newlyClickedTarget);
}


// --- Selection and Highlighting Logic ---

/**
 * Manages the selection of an object, including highlighting and isolation.
 * @param {THREE.Object3D | null} target - The object to select, or null to deselect.
 */
function handleSelection(target) {
    removeAllHighlights();
    deIsolateAllObjects();

    let newSelection = null;
    // Select the new target if it's different from the current selection
    if (target && (!state.selectedObjectOrGroup || state.selectedObjectOrGroup.uuid !== target.uuid)) {
        newSelection = target;
    }

    state.selectedObjectOrGroup = newSelection;

    if (state.selectedObjectOrGroup) {
        applyHighlight(state.selectedObjectOrGroup, HIGHLIGHT_COLOR_SINGLE);
        zoomToAndIsolate(state.selectedObjectOrGroup);
    }

    updateTreeSelection();
    updateInfoPanel();
}

/**
 * Applies a highlight material to a target object.
 */
const applyHighlight = (target, color) => {
    if (!target) return;

    target.traverse(child => {
        if (child.isMesh && child.material) {
            if (!state.originalMeshMaterials.has(child.uuid)) {
                state.originalMeshMaterials.set(child.uuid, child.material);
            }

            const createHighlightMaterial = (originalMat) => {
                const highlightMat = originalMat.clone();
                if (highlightMat.color) highlightMat.color.set(color);
                else if (highlightMat.emissive) highlightMat.emissive.set(color);
                return highlightMat;
            };

            child.material = Array.isArray(child.material)
                ? child.material.map(createHighlightMaterial)
                : createHighlightMaterial(child.material);
        }
    });
};


/**
 * Restores original materials to all highlighted objects.
 */
const removeAllHighlights = () => {
    state.originalMeshMaterials.forEach((originalMaterial, meshUuid) => {
        const mesh = scene.getObjectByProperty('uuid', meshUuid);
        if (mesh && mesh.isMesh) {
            mesh.material = originalMaterial;
        }
    });
    state.originalMeshMaterials.clear();
};


/**
 * Isolates a target object by making other objects transparent.
 */
function zoomToAndIsolate(targetObject) {
    if (!targetObject) return;

    deIsolateAllObjects();
    state.isIsolateModeActive = true;

    // Zoom logic
    const box = new THREE.Box3().setFromObject(targetObject);
    if (box.isEmpty()) { state.isIsolateModeActive = false; return; }
    const center = box.getCenter(new THREE.Vector3());
    const sphere = box.getBoundingSphere(new THREE.Sphere());
    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    let distance = (sphere.radius / Math.sin(fovInRadians / 2)) * 1.5;
    const offsetDirection = camera.position.clone().sub(controls.target).normalize();
    if (offsetDirection.lengthSq() === 0) offsetDirection.set(0.5, 0.5, 1).normalize();
    camera.position.copy(center).addScaledVector(offsetDirection, distance);
    controls.target.copy(center);

    // Isolation logic
    state.loadedObjectModelRoot.traverse((object) => {
        if (!object.isMesh) return;

        let isPartOfSelected = false;
        object.traverseAncestors((ancestor) => {
            if (ancestor === targetObject) isPartOfSelected = true;
        });

        if (!isPartOfSelected) {
            if (!state.originalObjectPropertiesForIsolate.has(object.uuid)) {
                state.originalObjectPropertiesForIsolate.set(object.uuid, { material: object.material, visible: object.visible });
            }
            if (object.visible) {
                const makeTransparent = (mat) => {
                    const newMat = mat.clone();
                    newMat.transparent = true;
                    newMat.opacity = 0.1;
                    return newMat;
                };
                object.material = Array.isArray(object.material) ? object.material.map(makeTransparent) : makeTransparent(object.material);
            }
        }
    });
}


/**
 * Restores visibility and materials to all isolated objects.
 */
function deIsolateAllObjects() {
    if (!state.isIsolateModeActive) return;

    state.originalObjectPropertiesForIsolate.forEach((props, uuid) => {
        const object = scene.getObjectByProperty('uuid', uuid);
        if (object && object.isMesh) {
            object.material = props.material;
            object.visible = props.visible;
        }
    });

    state.originalObjectPropertiesForIsolate.clear();
    state.isIsolateModeActive = false;
}

// --- UI Update Functions ---

/**
 * Shows or hides the main loader overlay.
 */
function showLoader(show, text = 'Loading...') {
    if (ui.loaderContainer) {
        ui.loaderContainer.style.display = show ? 'flex' : 'none';
        ui.loaderText.textContent = text;
    }
}

/**
 * Updates the information panel with details of the selected object.
 */
function updateInfoPanel() {
    let headerInfo = `WSCenID: ${state.parsedWSCenID || "N/A"}\nPJNo: ${state.parsedPJNo || "N/A"}\n----\n`;
    if (!state.parsedWSCenID && !state.parsedPJNo) {
        headerInfo = `WSCenID: \nPJNo: \n----\n`;
    }

    if (state.selectedObjectOrGroup) {
        let rawName = state.selectedObjectOrGroup.name || "Unnamed";
        let displayName, displayId = "N/A";
        const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
        if (splitIndex > 0) {
            displayName = rawName.substring(0, splitIndex);
            displayId = rawName.substring(splitIndex + 1);
        } else {
            displayName = rawName;
        }

        let selectionInfo = `名前: ${displayName}\nID: ${displayId}\n`;
        const data = state.elementIdDataMap.get(displayId);
        if (data) {
            selectionInfo += `\nカテゴリー名: ${data['カテゴリー名'] || "N/A"}\nファミリ名: ${data['ファミリ名'] || "N/A"}\nタイプ_ID: ${data['タイプ_ID'] || "N/A"}`;
        } else {
            selectionInfo += '\nカテゴリー名: "" \nファミリ名: ""';
        }
        ui.objectInfoPanel.textContent = headerInfo + selectionInfo;
    } else {
        ui.objectInfoPanel.textContent = headerInfo + 'None Selected';
    }
}


/**
 * Updates the selection style in the model tree UI.
 */
function updateTreeSelection() {
    document.querySelectorAll('#modelTreePanel .tree-item.selected').forEach(el => el.classList.remove('selected'));
    if (state.selectedObjectOrGroup) {
        const treeItemDiv = document.querySelector(`#modelTreePanel li[data-uuid="${state.selectedObjectOrGroup.uuid}"] .tree-item`);
        if (treeItemDiv) {
            treeItemDiv.classList.add('selected');
        }
    }
}

// --- Data and Tree Building ---

async function parseObjHeader(filePath) {
    try {
        const response = await fetch(filePath);
        if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
        const text = await response.text();
        const firstLine = text.substring(0, text.indexOf('\n')).trim();

        if (firstLine.startsWith("# ")) {
            const content = firstLine.substring(2).trim();
            const match = content.match(/^([0-9a-fA-F]{8}(-[0-9a-fA-F]{4}){3}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
            if (match) {
                state.parsedWSCenID = match[1];
                state.parsedPJNo = match[2];
            }
        }
    } catch (error) {
        console.error("Error fetching or parsing OBJ header:", error);
    }
}

function getAllElementIdsFromModel(model) {
    const ids = new Set();
    model.traverse(child => {
        if (child.name && child.parent === model) {
            const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
            if (splitIndex > 0) {
                ids.add(child.name.substring(splitIndex + 1));
            }
        }
    });
    return Array.from(ids);
}

async function fetchAllCategoryData(wscenId, allElementIds) {
    if (allElementIds.length === 0) return;

    const batchSize = 900;
    for (let i = 0; i < allElementIds.length; i += batchSize) {
        const batch = allElementIds.slice(i, i + batchSize);
        showLoader(true, `Fetching Categories... (${i + batch.length}/${allElementIds.length})`);
        try {
            const data = await $.ajax({
                type: "post",
                url: API_URL,
                data: { _token: CSRF_TOKEN, WSCenID: wscenId, ElementIds: batch },
            });
            for (const elementId in data) {
                state.elementIdDataMap.set(elementId, data[elementId]);
            }
        } catch (err) {
            console.error(`Error fetching category data batch starting at index ${i}:`, err);
            throw err; // Propagate error to stop the loading process
        }
    }
}

function buildAndPopulateCategorizedTree() {
    showLoader(true, "Building model tree...");
    const categorizedObjects = {};

    state.loadedObjectModelRoot.traverse(child => {
        if ((child.isGroup || child.isMesh) && child.name && child.parent === state.loadedObjectModelRoot) {
            const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
            const displayId = splitIndex > 0 ? child.name.substring(splitIndex + 1) : null;
            const data = displayId ? state.elementIdDataMap.get(displayId) : null;
            const category = data ? data['カテゴリー名'] : (displayId ? "カテゴリー無し" : "名称未分類");

            if (!categorizedObjects[category]) categorizedObjects[category] = [];
            categorizedObjects[category].push(child);
        }
    });

    ui.modelTreeList.innerHTML = '';
    Object.keys(categorizedObjects).sort().forEach(categoryName => {
        createCategoryNode(categoryName, categorizedObjects[categoryName]);
    });
    ui.modelTreePanel.style.display = 'block';
}


function createCategoryNode(categoryName, objects) {
    const categoryLi = document.createElement('li');
    const itemContent = document.createElement('div');
    itemContent.className = 'tree-item';
    itemContent.style.fontWeight = 'bold';

    const toggler = document.createElement('span');
    toggler.className = 'toggler';
    toggler.textContent = '▼';

    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = `${categoryName} (${objects.length})`;

    const subList = document.createElement('ul');

    toggler.addEventListener('click', (e) => {
        e.stopPropagation();
        const isCollapsed = subList.style.display === 'none';
        subList.style.display = isCollapsed ? 'block' : 'none';
        toggler.textContent = isCollapsed ? '▼' : '▶';
    });

    itemContent.append(toggler, nameSpan);
    categoryLi.append(itemContent, subList);
    ui.modelTreeList.appendChild(categoryLi);

    objects.forEach(object => createObjectNode(object, subList, 1));
}

function createObjectNode(object, parentULElement, depth) {
    const listItem = document.createElement('li');
    listItem.dataset.uuid = object.uuid;

    const itemContent = document.createElement('div');
    itemContent.className = 'tree-item';
    itemContent.style.paddingLeft = `${depth * 15 + 10}px`;
    itemContent.addEventListener('click', () => handleSelection(object));

    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = object.name;
    nameSpan.title = object.name;
    
    const toggler = document.createElement('span');
    toggler.className = 'toggler empty-toggler';
    toggler.innerHTML = '&nbsp;'; // Non-breaking space for layout

    const visibilityToggle = document.createElement('span');
    visibilityToggle.className = 'visibility-toggle visible-icon';
    visibilityToggle.title = 'Hide';
    visibilityToggle.addEventListener('click', (e) => {
        e.stopPropagation();
        object.visible = !object.visible;
        visibilityToggle.classList.toggle('visible-icon', object.visible);
        visibilityToggle.classList.toggle('hidden-icon', !object.visible);
        visibilityToggle.title = object.visible ? 'Hide' : 'Show';
        if (!object.visible && state.selectedObjectOrGroup?.uuid === object.uuid) {
            handleSelection(null); // Deselect if hidden
        }
    });

    itemContent.append(toggler, nameSpan, visibilityToggle);
    listItem.appendChild(itemContent);
    parentULElement.appendChild(listItem);
}


// --- Utility Functions ---

function frameObject(objectToFrame) {
    const box = new THREE.Box3().setFromObject(objectToFrame);
    if (box.isEmpty()) return; // Don't frame an empty object

    const center = box.getCenter(new THREE.Vector3());
    const sphere = box.getBoundingSphere(new THREE.Sphere());
    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    const distance = (sphere.radius / Math.sin(fovInRadians / 2)) * 1.3; // 1.3 zoomOutFactor

    const cameraDirection = new THREE.Vector3(1, 0.6, 1).normalize();
    const newPosition = center.clone().addScaledVector(cameraDirection, distance);

    camera.position.copy(newPosition);
    camera.lookAt(center);
    controls.target.copy(center);
}


/**
 * Records user access history if the required elements are present.
 */
function recordAccessHistoryIfNeeded() {
    const loginUserId = document.getElementById('hidLoginID')?.value;
    if (loginUserId) {
        recordAccessHistory(loginUserId, "/DL_DWH.png", "DLDWH/objviewer", "OBJビューア");
    }
}

/**
 * The main animation loop.
 */
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.autoClearColor = false; // Important for gradient background
    renderer.render(scene, camera);
}


// --- Application Start ---
document.addEventListener('DOMContentLoaded', init);
