<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Psr7\Request as GuzzleRequest;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Exception;

class DLDWHDataObjectViewerController extends Controller
{
    public function objViewer()
    {
        return view('DLDWH.OBJViewer');
    }

    public function getCategoryNameByElementId(Request $request)
    {
        $WSCenID = $request->input('WSCenID');
        $elementIds = $request->input('ElementIds');
        $dldwhModle = new DLDHWDataImportModel();
        return $dldwhModle->getCategoryNameByElementId($WSCenID, $elementIds);
    }

    public function getProjectList(Request $request)
    {
        $mainFolderId = $request->input('folderId');
        if (session()->has('access_token')) {
            $access_token = session('access_token');
            if ($access_token == "") {
                return "no_token";
            }

            if (empty($mainFolderId) || empty($access_token)) {
                return response()->json(['error' => 'Folder ID and Access Token are required.'], 400);
            }

            try {
                $client = new Client(['verify' => false]);
                $requestURL = "https://api.box.com/2.0/folders/" . $mainFolderId . "/items?fields=id,name";
                $header = [
                    "Authorization" => "Bearer " . $access_token,
                    "Accept" => "application/json"
                ];

                $response = $client->request('GET', $requestURL, ['headers' => $header]);
                $items = json_decode($response->getBody()->getContents())->entries;

                $projects = [];
                foreach ($items as $item) {
                    if ($item->type == "folder") {
                        $projects[] = ['id' => $item->id, 'name' => $item->name];
                    }
                }
                return response()->json($projects);
            } catch (Exception $e) {
                return response()->json(['error' => "Failed to read project list from Box: " . $e->getMessage()], 500);
            }
        } else {
            return "no_token";
        }
    }

    public function getObjList(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        $accessToken = session('access_token');

        if (empty($projectFolderId) || empty($accessToken)) {
            return response()->json(['error' => 'Folder ID and Access Token are required.'], 400);
        }

        try {
            $client = new Client(['verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken, "Accept" => "application/json"];

            $limit = 1000;
            $offset = 0;
            $allFolderItems = [];

            do {
                $requestURL = "https://api.box.com/2.0/folders/" . $projectFolderId . "/items?fields=id,name&limit=" . $limit . "&offset=" . $offset;
                $response = $client->request('GET', $requestURL, ['headers' => $header]);
                $body = json_decode($response->getBody()->getContents());

                $allFolderItems = array_merge($allFolderItems, $body->entries);
                $offset += $limit;
            } while (count($allFolderItems) < $body->total_count);

            $objFiles = [];
            $mtlFile = null;

            foreach ($allFolderItems as $item) {
                if ($item->type == "file" && strtolower(pathinfo($item->name, PATHINFO_EXTENSION)) === 'mtl') {
                    $mtlFile = ['id' => $item->id, 'name' => $item->name];
                    break;
                }
            }

            if ($mtlFile === null) {
                return response()->json(['error' => 'No MTL file found in the specified folder.'], 404);
            }

            foreach ($allFolderItems as $item) {
                if ($item->type == "file" && strtolower(pathinfo($item->name, PATHINFO_EXTENSION)) === 'obj') {
                    $objFiles[] = ['id' => $item->id, 'name' => $item->name];
                }
            }

            return response()->json(['mtl' => $mtlFile, 'objs' => $objFiles]);
        } catch (Exception $e) {
            return response()->json(['error' => "Failed to read file list from Box: " . $e->getMessage()], 500);
        }
    }

    public function getDownloadUrls(Request $request)
    {
        $fileIds = $request->input('fileIds');
        $accessToken = session('access_token');

        if (empty($fileIds) || !is_array($fileIds) || empty($accessToken)) {
            return response()->json(['error' => 'File IDs array and Access Token are required.'], 400);
        }

        $client = new Client(['verify' => false]);
        $downloadUrls = [];

        $requests = function ($fileIds, $accessToken) {
            $header = [
                "Authorization" => "Bearer " . $accessToken,
                "Accept" => "application/json"
            ];
            foreach ($fileIds as $fileId) {
                $url = "https://api.box.com/2.0/files/" . $fileId . "?fields=download_url";
                yield $fileId => new GuzzleRequest('GET', $url, $header);
            }
        };

        $pool = new Pool($client, $requests($fileIds, $accessToken), [
            // **FIX**: Reduced concurrency to avoid hitting Box API rate limits (429 errors).
            'concurrency' => 10,
            'fulfilled' => function ($response, $fileId) use (&$downloadUrls) {
                $fileInfo = json_decode($response->getBody()->getContents());
                if (isset($fileInfo->download_url)) {
                    $downloadUrls[$fileId] = $fileInfo->download_url;
                }
            },
            'rejected' => function ($reason, $fileId) {
                Log::error("Failed to get download URL for file ID {$fileId}: " . $reason->getMessage());
            },
        ]);

        $promise = $pool->promise();
        $promise->wait();

        return response()->json($downloadUrls);
    }
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
const highlightColorSingle = new THREE.Color(0xa0c4ff);
const elementIdDataMap = new Map();

const BOX_MAIN_FOLDER_ID = "332324771912";

// --- Scene Setup ---
const scene = new THREE.Scene();
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
directionalLight.shadow.mapSize.width = 2048;
directionalLight.shadow.mapSize.height = 2048;
scene.add(directionalLight);
const hemiLight = new THREE.HemisphereLight(0xffffff, 0x8d8d8d, 1.5);
hemiLight.position.set(0, 50, 0);
scene.add(hemiLight);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;

var CSRF_TOKEN = $('meta[name="csrf-token"]').attr('content');

$(document).ready(function () {
    var login_user_id = $("#hidLoginID").val();
    var img_src = "/DL_DWH.png";
    var url = "DLDWH/objviewer";
    var content_name = "OBJビューア";
    recordAccessHistory(login_user_id, img_src, url, content_name);
    onWindowResize();
    populateProjectDropdown();
    animate();
});

async function populateProjectDropdown() {
    try {
        const projects = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getProjectList",
            data: { _token: CSRF_TOKEN, folderId: BOX_MAIN_FOLDER_ID },
        });
        modelSelector.innerHTML = '';
        if (projects.includes("no_token")) {
            alert("BOXにログインされていないためobjファイルを取得きませんでした。");
        } else if (projects && projects.length > 0) {
            projects.forEach(project => {
                const option = document.createElement('option');
                option.value = project.id;
                option.textContent = project.name;
                modelSelector.appendChild(option);
            });
            loadModel(projects[2].id, projects[2].name);
        } else {
            if (loaderTextElement) loaderTextElement.textContent = "No projects found.";
        }
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "Error fetching project list.";
    }
}

async function loadModel(projectFolderId, projectName) {
    if (loadedObjectModelRoot) scene.remove(loadedObjectModelRoot);
    loadedObjectModelRoot = null;
    selectedObjectOrGroup = null;
    originalMeshMaterials.clear();
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
    elementIdDataMap.clear();
    modelTreeList.innerHTML = '';
    if (modelTreePanel) modelTreePanel.style.display = 'none';
    parsedWSCenID = "";
    parsedPJNo = "";

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list for ${projectName}...`;

        const fileList = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getObjList",
            data: { _token: CSRF_TOKEN, folderId: projectFolderId },
        });

        if (!fileList || !fileList.mtl || !fileList.objs || fileList.objs.length === 0) {
            throw new Error(`Incomplete file list for project "${projectName}".`);
        }

        const mtlFileInfo = fileList.mtl;
        const objFileInfoList = fileList.objs;
        const allFileIds = [mtlFileInfo.id, ...objFileInfoList.map(f => f.id)];

        if (loaderTextElement) loaderTextElement.textContent = `Preparing secure downloads...`;
        const downloadUrlMap = {};
        const batchSize = 900;

        for (let i = 0; i < allFileIds.length; i += batchSize) {
            const batch = allFileIds.slice(i, i + batchSize);
            if (loaderTextElement) {
                loaderTextElement.textContent = `Preparing secure downloads... (${i + batch.length}/${allFileIds.length})`;
            }
            const batchUrlMap = await $.ajax({
                type: "post",
                url: url_prefix + "/box/getDownloadUrls",
                data: { _token: window.CSRF_TOKEN, fileIds: batch },
            });
            Object.assign(downloadUrlMap, batchUrlMap);
        }

        if (loaderTextElement) loaderTextElement.textContent = `Loading Materials...`;
        const mtlUrl = downloadUrlMap[mtlFileInfo.id];
        if (!mtlUrl) throw new Error(`Could not get download URL for MTL file ${mtlFileInfo.name}`);

        const mtlLoader = new MTLLoader();
        const materialsCreator = await mtlLoader.loadAsync(mtlUrl);
        materialsCreator.preload();

        if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (0/${objFileInfoList.length})...`;
        const CONCURRENT_DOWNLOADS = 10;
        const allObjContents = [];
        let downloadedCount = 0;
        const downloadQueue = [...objFileInfoList];

        const downloadBatch = async () => {
            const currentBatchPromises = [];
            while (downloadQueue.length > 0 && currentBatchPromises.length < CONCURRENT_DOWNLOADS) {
                const objInfo = downloadQueue.shift();
                const objUrl = downloadUrlMap[objInfo.id];
                
                // **FIX**: Add safety check. Only proceed if we have a valid URL.
                if (objInfo && objUrl) {
                    const promise = fetch(objUrl).then(res => {
                        if (!res.ok) throw new Error(`Failed to download ${objInfo.name}: ${res.statusText}`);
                        return res.text();
                    }).then(content => {
                        downloadedCount++;
                        if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (${downloadedCount}/${objFileInfoList.length})...`;
                        return { content, info: objInfo };
                    });
                    currentBatchPromises.push(promise);
                } else {
                    console.warn(`Skipping file ${objInfo.name} because its download URL could not be retrieved.`);
                    downloadedCount++; // Still increment to show progress
                    if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (${downloadedCount}/${objFileInfoList.length})...`;
                }
            }
            const results = await Promise.all(currentBatchPromises);
            // Filter out any potential null/undefined results from skipped files
            allObjContents.push(...results.filter(Boolean));
            if (downloadQueue.length > 0) await downloadBatch();
        };
        await downloadBatch();

        if (allObjContents.length > 0) {
            const headerData = await parseObjHeader(allObjContents[0].content);
            if (headerData) {
                parsedWSCenID = headerData.wscenId;
                parsedPJNo = headerData.pjNo;
            }
        }

        if (loaderTextElement) loaderTextElement.textContent = `Processing geometry...`;
        const objLoader = new OBJLoader();
        objLoader.setMaterials(materialsCreator);

        const loadedObjects = allObjContents.map(objData => objLoader.parse(objData.content));

        const combinedModel = new THREE.Group();
        loadedObjects.forEach(object => {
            while (object.children.length > 0) {
                combinedModel.add(object.children[0]);
            }
        });
        loadedObjectModelRoot = combinedModel;

        const initialBox = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        const initialCenter = initialBox.getCenter(new THREE.Vector3());
        loadedObjectModelRoot.position.sub(initialCenter);
        const scaledBox = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        const maxDim = Math.max(scaledBox.getSize(new THREE.Vector3()).x, scaledBox.getSize(new THREE.Vector3()).y, scaledBox.getSize(new THREE.Vector3()).z);
        const desiredMaxDimension = 150;
        if (maxDim > 0) {
            const scale = desiredMaxDimension / maxDim;
            loadedObjectModelRoot.scale.set(scale, scale, scale);
        }
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;

        const allIds = [];
        loadedObjectModelRoot.traverse(child => {
            if (child.name) {
                const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                if (splitIndex > 0) allIds.push(child.name.substring(splitIndex + 1));
            }
        });
        await fetchAllCategoryData(parsedWSCenID, [...new Set(allIds)]);

        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        if (loaderContainer) loaderContainer.style.display = 'none';
        updateInfoPanel();

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error);
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model. Check console.`;
    }
}

async function parseObjHeader(objContent) {
    try {
        const lines = objContent.split(/\r?\n/);
        if (lines.length > 0) {
            const firstLine = lines[0].trim();
            if (firstLine.startsWith("# ")) {
                const content = firstLine.substring(2).trim();
                const pattern1Match = content.match(/^([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
                if (pattern1Match) return { wscenId: pattern1Match[1], pjNo: pattern1Match[2] };
                if (content.includes("ワークシェアリングされてない")) return { wscenId: "", pjNo: "" };
            }
        }
    } catch (error) {
        console.error("Error parsing OBJ header:", error);
    }
    return { wscenId: null, pjNo: null };
}

async function fetchAllCategoryData(wscenId, allElementIds) {
    if (allElementIds.length === 0) return;
    const batchSize = 900;
    for (let i = 0; i < allElementIds.length; i += batchSize) {
        const batch = allElementIds.slice(i, i + batchSize);
        if (loaderTextElement) loaderTextElement.textContent = `Fetching Categories... (${i + batch.length}/${allElementIds.length})`;
        const data = await $.ajax({
            type: "post",
            url: url_prefix + "/DLDWH/getDatas",
            data: { _token: CSRF_TOKEN, WSCenID: wscenId, ElementIds: batch },
        });
        for (const elementId in data) {
            elementIdDataMap.set(elementId, data[elementId]);
        }
    }
}

async function buildAndPopulateCategorizedTree() {
    if (!loadedObjectModelRoot || !modelTreeList) return;
    if (loaderTextElement) loaderTextElement.textContent = "Building model tree...";
    const categorizedObjects = {};
    loadedObjectModelRoot.traverse(child => {
        if (child.name && child.parent === loadedObjectModelRoot) {
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
    const finalDistance = distance * 1.3;
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
    toggler.innerHTML = ' ';
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
    target.traverse(child => {
        if (child.isMesh && child.material) {
            if (!originalMeshMaterials.has(child.uuid)) {
                originalMeshMaterials.set(child.uuid, child.material);
            }
            if (Array.isArray(child.material)) {
                child.material = child.material.map(mat => mat.clone());
                child.material.forEach(mat => mat.color.set(color));
            } else {
                child.material = child.material.clone();
                child.material.color.set(color);
            }
        }
    });
};

const removeAllHighlights = () => {
    originalMeshMaterials.forEach((originalMaterial, uuid) => {
        const mesh = scene.getObjectByProperty('uuid', uuid);
        if (mesh) mesh.material = originalMaterial;
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
    let headerInfo = `WSCenID: ${parsedWSCenID || "N/A"}\nPJNo: ${parsedPJNo || "N/A"}\n----\n`;
    if (parsedWSCenID === "" && parsedPJNo === "") headerInfo = `WSCenID: \nPJNo: \n----\n`;
    if (selectedObjectOrGroup) {
        let rawName = selectedObjectOrGroup.name || "Unnamed";
        let displayId = "N/A";
        const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
        if (splitIndex > 0) displayId = rawName.substring(splitIndex + 1);
        let selectionInfo = `ID: ${displayId}\n`;
        if (displayId && elementIdDataMap.has(displayId)) {
            const data = elementIdDataMap.get(displayId);
            selectionInfo += `カテゴリー名: ${data['カテゴリー名'] || "N/A"}\nファミリ名: ${data['ファミリ名'] || "N/A"}`;
        }
        objectInfoPanel.textContent = headerInfo + selectionInfo;
    } else {
        objectInfoPanel.textContent = headerInfo + 'None Selected';
    }
}

window.addEventListener('click', (event) => {
    if (!loadedObjectModelRoot || event.target.closest('#modelTreePanel')) return;
    const rect = renderer.domElement.getBoundingClientRect();
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);
    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let current = intersects[0].object;
        while (current && current.parent !== loadedObjectModelRoot) {
            current = current.parent;
        }
        if (current) newlyClickedTarget = current;
    }
    handleSelection(newlyClickedTarget);
});

if (closeModelTreeBtn) closeModelTreeBtn.addEventListener('click', () => { if (modelTreePanel) modelTreePanel.style.display = 'none'; });

if (modelTreeSearch) {
    modelTreeSearch.addEventListener('input', (e) => {
        const searchTerm = e.target.value.toLowerCase().trim();
        document.querySelectorAll('#modelTreeList li').forEach(li => {
            const nameSpan = li.querySelector('.group-name');
            if (nameSpan) {
                const isMatch = nameSpan.textContent.toLowerCase().includes(searchTerm);
                li.style.display = isMatch ? '' : 'none';
                if (isMatch) {
                    let parent = li.parentElement.closest('li');
                    while (parent) {
                        parent.style.display = '';
                        const subList = parent.querySelector('ul');
                        if (subList) subList.style.display = 'block';
                        const toggler = parent.querySelector('.toggler:not(.empty-toggler)');
                        if (toggler) toggler.textContent = '▼';
                        parent = parent.parentElement.closest('li');
                    }
                }
            }
        });
    });
}

if (toggleUiButton) {
    toggleUiButton.addEventListener('click', () => {
        const isVisible = modelTreePanel.style.display !== 'none';
        if (modelTreePanel) modelTreePanel.style.display = isVisible ? 'none' : 'block';
        toggleUiButton.textContent = isVisible ? '📊' : '❌';
        toggleUiButton.title = isVisible ? "Show UI Panels" : "Hide UI Panels";
    });
}

function onWindowResize() {
    const { clientWidth, clientHeight } = viewerContainer;
    camera.aspect = clientWidth / clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(clientWidth, clientHeight);
}
window.addEventListener('resize', onWindowResize);

if (modelSelector) {
    modelSelector.addEventListener('change', (event) => {
        const selectedId = event.target.value;
        const selectedName = event.target.options[event.target.selectedIndex].text;
        loadModel(selectedId, selectedName);
    });
}

function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.autoClearColor = false;
    renderer.render(scene, camera);
}
