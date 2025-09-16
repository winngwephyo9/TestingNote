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
let syncPollingInterval = null;

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

async function loadModel(projectFolderId, projectName) {
    if (syncPollingInterval) {
        clearInterval(syncPollingInterval);
        syncPollingInterval = null;
    }
    
    resetScene();
    if (loaderContainer) loaderContainer.style.display = 'flex';

    try {
        if (loaderTextElement) loaderTextElement.textContent = `Fetching model data for ${projectName}...`;
        
        const response = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getModelData",
            data: { _token: CSRF_TOKEN, folderId: projectFolderId }
        });

        const filePairs = response.objMtlPairs; 
        const syncStatus = response.sync_status;

        if (syncStatus === 'processing') {
            if (loaderTextElement) loaderTextElement.textContent = `サーバーで大規模なモデル同期中です... この処理には数十分かかる場合があります。`;
            if (filePairs && filePairs.length > 0) {
                if (loaderTextElement) loaderTextElement.textContent += ` (現在、前回同期したデータを表示しています)`;
            } else {
                startSyncStatusPolling(projectFolderId, projectName);
                return; 
            }
        }
        if (syncStatus === 'failed') {
            if (loaderTextElement) loaderTextElement.textContent = `前回の同期に失敗しました。管理者にご確認ください。`;
        }
        
        if (!Array.isArray(filePairs) || filePairs.length === 0) {
            if (syncStatus !== 'processing') {
                throw new Error(response.error || `No model data found for project "${projectName}".`);
            }
            return;
        }
        
        if (loaderTextElement) {
            if(syncStatus !== 'processing') {
                loaderTextElement.textContent = `Processing geometry and materials...`;
            }
        }
        
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

        const box = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        const center = box.getCenter(new THREE.Vector3());
        loadedObjectModelRoot.position.sub(center);
        const size = box.getSize(new THREE.Vector3());
        const maxDim = Math.max(size.x, size.y, size.z);
        const scale = (maxDim > 0) ? (150 / maxDim) : 1;
        loadedObjectModelRoot.scale.set(scale, scale, scale);
        loadedObjectModelRoot.userData.modelScale = scale;
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;

        const allIds = [...new Set(
            loadedObjectModelRoot.children
                .map(child => {
                    const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                    return splitIndex > 0 ? child.name.substring(splitIndex + 1) : null;
                })
                .filter(Boolean)
        )];
        await fetchAllCategoryData(parsedWSCenID, allIds);
        
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        updateInfoPanel();
        
        if(syncStatus === 'processing'){
            startSyncStatusPolling(projectFolderId, projectName);
        }

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error.responseText || error);
        let errorMessage = `Error: ${error.message || 'An unknown error occurred'}.`;
        if (error.responseJSON && error.responseJSON.error) {
            errorMessage = `Error: ${error.responseJSON.error}`;
        }
        if (loaderTextElement) loaderTextElement.textContent = errorMessage;
    } finally {
        if (!syncPollingInterval) {
            if (loaderContainer) loaderContainer.style.display = 'none';
        }
    }
}

function startSyncStatusPolling(projectFolderId, projectName) {
    syncPollingInterval = setInterval(async () => {
        try {
            const response = await $.ajax({
                type: "post",
                url: url_prefix + "/box/getSyncStatus",
                data: { _token: CSRF_TOKEN, folderId: projectFolderId }
            });

            if (response.sync_status === 'completed' || response.sync_status === 'failed') {
                clearInterval(syncPollingInterval);
                syncPollingInterval = null;
                if (loaderTextElement) loaderTextElement.textContent = `同期が完了しました。最新のモデルを読み込みます...`;
                loadModel(projectFolderId, projectName);
            }
        } catch (error) {
            console.error("Failed to get sync status:", error);
            clearInterval(syncPollingInterval);
            syncPollingInterval = null;
        }
    }, 10000); // 10秒ごと
}


// ... (これ以降のヘルパー関数はすべて変更ありません) ...
// resetScene, parseObjHeader, fetchAllCategoryData, buildAndPopulateCategorizedTree, frameObject,
// createCategoryNode, createObjectNode, handleSelection, applyHighlight, removeAllHighlights,
// zoomToAndIsolate, deIsolateAllObjects, updateInfoPanel, イベントリスナー, animate,
// handleResetView, getVolumeOfSelectedObject, calculateMeshVolume など
