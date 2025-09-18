// 4. 【新規】同期ジョブを開始させるためのAPI
Route::post('/box/startSync', [DLDWHDataObjectViewerController::class, 'startSync']);

// 5. 【新規】バックグラウンドジョブの状態を問い合わせるためのポーリング用API
Route::post('/box/checkSyncJobStatus', [DLDWHDataObjectViewerController::class, 'checkSyncJobStatus']);



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
let loadedObjectModelRoot = null;
let selectedObjectOrGroup = null;
let parsedWSCenID = "";
let parsedPJNo = "";
const originalMeshMaterials = new Map();
const originalObjectPropertiesForIsolate = new Map();
let isIsolateModeActive = false;
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
let mouseDownPosition = new THREE.Vector2();
const highlightColorSingle = new THREE.Color(0xa0c4ff);
const elementIdDataMap = new Map();
let currentLoadedProjectId = null;

const BOX_MAIN_FOLDER_ID = "339110566808";
var CSRF_TOKEN = $('meta[name="csrf-token"]').attr('content');

// --- Scene Setup ---
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 0.1, 20000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
const controls = new OrbitControls(camera, renderer.domElement);
function setupScene() {
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

    camera.position.set(10, 10, 10);
    camera.lookAt(0, 0, 0);

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

    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
}

// --- Main Application Logic ---
$(document).ready(function () {
    const login_user_id = $("#hidLoginID").val();
    if (typeof recordAccessHistory === 'function') {
        recordAccessHistory(login_user_id, "/DL_DWH.png", "DLDWH/objviewer", "OBJビューア");
    }
    setupScene();
    onWindowResize();
    initiateProjectPopulation();
    animate();

    // Event Listeners
    if (resetViewButton) resetViewButton.addEventListener('click', handleResetView);
    if (modelSelector) modelSelector.addEventListener('change', modelSelectorChanged);
    viewerContainer.addEventListener('mousedown', (e) => mouseDownPosition.set(e.clientX, e.clientY), false);
    viewerContainer.addEventListener('mouseup', handleMouseClick, false);
    window.addEventListener('resize', onWindowResize);
});

async function initiateProjectPopulation() {
    try {
        const response = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getProjectList",
            data: { _token: CSRF_TOKEN, folderId: BOX_MAIN_FOLDER_ID }
        });

        const projects = response.projects;
        modelSelector.innerHTML = '';

        if (Array.isArray(projects) && projects.length > 0) {
            projects.forEach(project => {
                const option = document.createElement('option');
                option.value = project.id;
                option.textContent = project.name;
                modelSelector.appendChild(option);
            });
            const modelToLoad = projects[0];
            startSync(modelToLoad.id, modelToLoad.name);
        } else {
            if (loaderTextElement) loaderTextElement.textContent = "表示できるプロジェクトがありません。";
            hideLoading();
        }
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "プロジェクトリストの取得中にエラーが発生しました。";
        hideLoading();
    }
}

function modelSelectorChanged() {
    const selectedId = modelSelector.value;
    const selectedName = modelSelector.options[modelSelector.selectedIndex].text;
    startSync(selectedId, selectedName);
}

/**
 * サーバーに同期ジョブの開始をリクエストする
 */
function startSync(projectFolderId, projectName) {
    if (currentLoadedProjectId === projectFolderId) return;
    currentLoadedProjectId = projectFolderId;

    showLoading();
    if (loaderTextElement) loaderTextElement.textContent = `サーバーと同期を開始しています...`;
    
    $.ajax({
        type: "post",
        url: url_prefix + "/box/startSync",
        data: { _token: CSRF_TOKEN, folderId: projectFolderId },
        success: function (response) {
            if (response.token === "no_token") {
                alert("BOXにログインされていないため更新できません。キャッシュされたデータを表示します。");
                fetchAndRenderModel(projectFolderId, projectName); 
            } else if (response.status === 'job_dispatched') {
                if (loaderTextElement) loaderTextElement.textContent = `サーバーで同期中です... この処理には数分かかる場合があります。`;
                checkSyncStatus(projectFolderId, projectName);
            }
        },
        error: function(err) {
            alert("同期の開始に失敗しました。管理者に問い合わせてください。");
            hideLoading();
        }
    });
}

/**
 * 同期ジョブの状態をポーリング（定期確認）する
 */
function checkSyncStatus(projectFolderId, projectName) {
    // 5秒後に実行
    setTimeout(async () => {
        // ユーザーが別のモデルを選択していたら、このポーリングは停止
        if (modelSelector.value !== projectFolderId) {
            console.log(`Polling for ${projectName} stopped because another project was selected.`);
            return;
        }

        try {
            const response = await $.ajax({
                type: "post",
                url: url_prefix + "/box/checkSyncJobStatus",
                data: { _token: CSRF_TOKEN, folderId: projectFolderId }
            });

            if (response.status === 'completed' || response.status === 'no_files_to_update') {
                const message = response.status === 'completed' ? '同期が完了しました。' : 'データは最新です。';
                if (loaderTextElement) loaderTextElement.textContent = `${message} モデルを読み込みます...`;
                fetchAndRenderModel(projectFolderId, projectName);
            } else if (response.status === 'failed') {
                alert(`同期に失敗しました: ${response.message}`);
                fetchAndRenderModel(projectFolderId, projectName); // 失敗してもキャッシュ表示を試みる
            } else { // 'processing'
                if (loaderTextElement) loaderTextElement.textContent += "."; // 進捗中であることを示す
                checkSyncStatus(projectFolderId, projectName); // ポーリングを続ける
            }
        } catch (error) {
            alert("同期状態の確認に失敗しました。管理者に問い合わせてください。");
            hideLoading();
        }
    }, 5000);
}

/**
 * 最終的なモデルデータをサーバーから取得し、3Dシーンを構築・描画する
 */
async function fetchAndRenderModel(projectFolderId, projectName) {
    resetScene(); // 描画前にシーンをクリア
    showLoading(); // ローディングを確実に表示
    if (loaderTextElement) loaderTextElement.textContent = `モデルデータを取得しています...`;

    try {
        const filePairs = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getModelData",
            data: { _token: CSRF_TOKEN, folderId: projectFolderId }
        });

        if (!Array.isArray(filePairs) || filePairs.length === 0) {
            throw new Error(`表示できるモデルデータがありません。`);
        }

        if (loaderTextElement) loaderTextElement.textContent = `モデルを構築しています...`;
        
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

        loadedObjectModelRoot = new THREE.Group();
        const validationBox = new THREE.Box3();
        allLoadedObjects.forEach(parsedGroup => {
            if (parsedGroup.isGroup) {
                while(parsedGroup.children.length > 0) {
                    const child = parsedGroup.children[0];
                    if (child.isMesh && child.geometry) {
                        try {
                            validationBox.setFromObject(child);
                            loadedObjectModelRoot.add(child);
                        } catch (e) {
                            console.warn("Discarding object with invalid geometry:", { name: child.name });
                        }
                    }
                }
            }
        });
        
        if (loadedObjectModelRoot.children.length === 0) {
            throw new Error("有効なオブジェクトを読み込めませんでした。");
        }

        const box = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        const center = box.getCenter(new THREE.Vector3());
        loadedObjectModelRoot.position.sub(center);
        const size = box.getSize(new THREE.Vector3());
        const maxDim = Math.max(size.x, size.y, size.z);
        const scale = (maxDim > 0) ? (150 / maxDim) : 1;
        loadedObjectModelRoot.scale.set(scale, scale, scale);
        loadedObjectModelRoot.userData.modelScale = scale;
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;

        const allIds = [...new Set(loadedObjectModelRoot.children.map(c => {
            const s = Math.max(c.name.lastIndexOf('_'), c.name.lastIndexOf('＿'));
            return s > 0 ? c.name.substring(s + 1) : null;
        }).filter(Boolean))];
        
        await fetchAllCategoryData(parsedWSCenID, allIds);
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        updateInfoPanel();
        
    } catch (error) {
        console.error("Failed to render model:", error);
        if (loaderTextElement) loaderTextElement.textContent = `モデルの表示に失敗しました: ${error.message}`;
    } finally {
        hideLoading();
    }
}


function showLoading() {
    if (loaderContainer) loaderContainer.style.display = 'flex';
}
function hideLoading() {
    if (loaderContainer) loaderContainer.style.display = 'none';
}

function resetScene() {
    if (loadedObjectModelRoot) {
        scene.remove(loadedObjectModelRoot);
        // ... (メモリ解放処理)
    }
    loadedObjectModelRoot = null;
    selectedObjectOrGroup = null;
    parsedWSCenID = "";
    parsedPJNo = "";
    originalMeshMaterials.clear();
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
    elementIdDataMap.clear();
    modelTreeList.innerHTML = '';
    updateInfoPanel();
    currentLoadedProjectId = null;
}

function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}

// ... parseObjHeader, fetchAllCategoryData, buildAndPopulateCategorizedTree, frameObject,
// createCategoryNode, createObjectNode, handleSelection, applyHighlight, removeAllHighlights,
// zoomToAndIsolate, deIsolateAllObjects, updateInfoPanel, イベントリスナー(handleMouseClick),
// onWindowResize, handleResetView, getVolumeOfSelectedObject, calculateMeshVolume などの
// ヘルパー関数は全て、以前の回答で完成したものをここに含めます。
