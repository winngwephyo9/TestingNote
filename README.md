
<?php

namespace App\Http\Controllers;

use App\Models\DLDHWDataImportModel;
use GuzzleHttp\Client;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Exception;
use GuzzleHttp\Exception\ClientException;

class DLDWHDataObjectViewerController extends Controller
{
    public function objViewer()
    {
        return view('DLDWH.OBJViewer');
    }

    /**
     * Get category name by elementId.
     * This function is correct. The client-side JS now calls this in batches,
     * which prevents the "max_input_vars" error.
     *
     * @param Request $request
     * @return mixed
     */
    public function getCategoryNameByElementId(Request $request)
    {
        $WSCenID = $request->input('WSCenID');
        $elementIds = $request->input('ElementIds'); // Expects a batch of IDs
        $dldwhModle = new DLDHWDataImportModel();
        return $dldwhModle->getCategoryNameByElementId($WSCenID, $elementIds);
    }

    /**
     * Lists the sub-folders (projects) within the main Box folder.
     *
     * @param Request $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function getProjectList(Request $request)
    {
        $mainFolderId = $request->input('folderId');
        if (!session()->has('access_token') || empty(session('access_token'))) {
            return response()->json(["no_token"]);
        }
        $access_token = session('access_token');

        if (empty($mainFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }

        try {
            $client = new Client(['verify' => false]);
            $requestURL = "https://api.box.com/2.0/folders/{$mainFolderId}/items?fields=id,name";
            $header = ["Authorization" => "Bearer " . $access_token];

            $response = $client->get($requestURL, ['headers' => $header]);
            $items = json_decode($response->getBody()->getContents())->entries;

            $projects = [];
            foreach ($items as $item) {
                if ($item->type == "folder") {
                    $projects[] = ['id' => $item->id, 'name' => $item->name];
                }
            }

            return response()->json($projects);
        } catch (Exception $e) {
            Log::error("Box API Error (getProjectList): " . $e->getMessage());
            return response()->json(['error' => "Failed to read project list from Box."], 500);
        }
    }

    /**
     * Lists all OBJ and MTL file IDs and names within a project folder, handling pagination.
     *
     * @param Request $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function getObjList(Request $request)
    {
        $projectFolderId = $request->input('folderId');
        if (!session()->has('access_token') || empty(session('access_token'))) {
            return response()->json(['error' => 'Access token is missing.'], 401);
        }
        $accessToken = session('access_token');

        if (empty($projectFolderId)) {
            return response()->json(['error' => 'Folder ID is required.'], 400);
        }

        try {
            $client = new Client(['verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken];
            $allFolderItems = [];
            $offset = 0;
            $limit = 1000; // Max limit per page for Box API

            do {
                $requestURL = "https://api.box.com/2.0/folders/{$projectFolderId}/items?fields=id,name&limit={$limit}&offset={$offset}";
                $response = $client->get($requestURL, ['headers' => $header]);
                $body = json_decode($response->getBody()->getContents());
                
                if (isset($body->entries)) {
                    $allFolderItems = array_merge($allFolderItems, $body->entries);
                    $offset += count($body->entries);
                } else {
                    // Break if entries are not present
                    break;
                }
                
            } while ($offset < $body->total_count);

            $objFiles = [];
            $mtlFile = null;

            foreach ($allFolderItems as $item) {
                if ($item->type == "file") {
                    $extension = strtolower(pathinfo($item->name, PATHINFO_EXTENSION));
                    if ($extension === 'mtl' && $mtlFile === null) {
                        $mtlFile = ['id' => $item->id, 'name' => $item->name];
                    } elseif ($extension === 'obj') {
                        $objFiles[] = ['id' => $item->id, 'name' => $item->name];
                    }
                }
            }

            if ($mtlFile === null) {
                return response()->json(['error' => 'No MTL file found in the specified folder.'], 404);
            }

            return response()->json(['mtl' => $mtlFile, 'objs' => $objFiles]);
        } catch (Exception $e) {
            Log::error("Box API Error (getObjList): " . $e->getMessage());
            return response()->json(['error' => "Failed to read file list from Box."], 500);
        }
    }

    /**
     * **NEW, EFFICIENT FUNCTION**
     * Gets temporary, pre-authenticated download URLs for multiple files from Box.
     * This allows the client to download files directly, solving the timeout issue.
     *
     * @param Request $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function getDownloadUrls(Request $request)
    {
        $fileIds = $request->input('fileIds'); // Expecting an array
        if (!session()->has('access_token') || empty(session('access_token'))) {
            return response()->json(['error' => 'Access token is missing.'], 401);
        }
        $accessToken = session('access_token');

        if (empty($fileIds) || !is_array($fileIds)) {
            return response()->json(['error' => 'File IDs array is required.'], 400);
        }

        try {
            $client = new Client(['verify' => false]);
            $header = ["Authorization" => "Bearer " . $accessToken];
            $downloadUrls = [];

            // This must be done in a loop as Box API does not support batching this request.
            // However, it's very fast as it only fetches metadata.
            foreach ($fileIds as $fileId) {
                // The '?fields=download_url' is an optimization to only get the URL
                $requestURL = "https://api.box.com/2.0/files/{$fileId}?fields=download_url";
                try {
                    $response = $client->get($requestURL, ['headers' => $header]);
                    $fileInfo = json_decode($response->getBody()->getContents());
                    if (isset($fileInfo->download_url)) {
                        $downloadUrls[$fileId] = $fileInfo->download_url;
                    }
                } catch (ClientException $e) {
                    // Log the error for a specific file but continue for others
                    Log::warning("Box API: Could not get download URL for file ID {$fileId}: " . $e->getResponse()->getBody()->getContents());
                }
            }
            return response()->json($downloadUrls);
        } catch (Exception $e) {
            Log::error("Box API Error (getDownloadUrls): " . $e->getMessage());
            return response()->json(['error' => "An error occurred while generating download URLs."], 500);
        }
    }

    /*
     * DEPRECATED: This function is the source of the timeout error and is no longer needed.
     * It has been replaced by the much more efficient `getDownloadUrls` method.
     
    public function getFileContents(Request $request)
    {
        // ... old, slow code removed ...
    }
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
var CSRF_TOKEN = $('meta[name="csrf-token"]').attr('content');

// --- Scene Setup, Camera, Renderer, Lighting, Controls ---
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

// --- Main Application Logic ---

$(document).ready(function () {
    const login_user_id = $("#hidLoginID").val();
    if(typeof recordAccessHistory === 'function') {
      recordAccessHistory(login_user_id, "/DL_DWH.png", "DLDWH/objviewer", "OBJビューア");
    }
    onWindowResize();
    populateProjectDropdown();
    animate();
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
            alert("BOXにログインされていないためobjファイルを取得きませんでした。");
            if (loaderTextElement) loaderTextElement.textContent = "Box login required.";
        } else if (projects && projects.length > 0) {
            projects.forEach(project => {
                const option = document.createElement('option');
                option.value = project.id;
                option.textContent = project.name;
                modelSelector.appendChild(option);
            });
            // Automatically load a default model (e.g., the third one if available, otherwise the first)
            const modelToLoad = projects.length > 2 ? projects[2] : projects[0];
            loadModel(modelToLoad.id, modelToLoad.name);
        } else {
            if (loaderTextElement) loaderTextElement.textContent = "No projects found in the main folder.";
        }
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "Error fetching project list from Box.";
    }
}

function resetScene() {
    if (loadedObjectModelRoot) {
        // Proper disposal of geometry and materials
        loadedObjectModelRoot.traverse(object => {
            if (object.isMesh) {
                if (object.geometry) object.geometry.dispose();
                if (Array.isArray(object.material)) {
                    object.material.forEach(material => material.dispose());
                } else if (object.material) {
                    object.material.dispose();
                }
            }
        });
        scene.remove(loadedObjectModelRoot);
    }
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
    updateInfoPanel();
}

/**
 * **REFACTORED for PERFORMANCE**
 * Loads a model using a fast, direct-from-Box download strategy.
 */
async function loadModel(projectFolderId, projectName) {
    resetScene();
    if (loaderContainer) loaderContainer.style.display = 'flex';

    try {
        // --- STEP 1: Get the list of file IDs from our server ---
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list for ${projectName}...`;
        const fileList = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getObjList",
            data: { _token: CSRF_TOKEN, folderId: projectFolderId }
        });
        if (!fileList || !fileList.mtl || !fileList.objs || fileList.objs.length === 0) {
            throw new Error(`Incomplete or empty file list for project "${projectName}".`);
        }

        // --- STEP 2: Get secure, temporary download URLs from our server ---
        // This is a quick metadata request.
        if (loaderTextElement) loaderTextElement.textContent = `Preparing secure downloads...`;
        const allFileIds = [fileList.mtl.id, ...fileList.objs.map(f => f.id)];
        const downloadUrlMap = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getDownloadUrls",
            data: { _token: CSRF_TOKEN, fileIds: allFileIds }
        });

        // --- STEP 3: Load the single MTL file *directly from Box* ---
        if (loaderTextElement) loaderTextElement.textContent = `Loading materials...`;
        const mtlUrl = downloadUrlMap[fileList.mtl.id];
        if (!mtlUrl) throw new Error(`Could not get download URL for the MTL file.`);
        const mtlLoader = new MTLLoader();
        const materialsCreator = await mtlLoader.loadAsync(mtlUrl);
        materialsCreator.preload();

        // --- STEP 4: Download all OBJ files in parallel, *directly from Box* ---
        // This is the key performance improvement that prevents server timeouts.
        const objFileInfoList = fileList.objs;
        let downloadedCount = 0;
        const totalFiles = objFileInfoList.length;
        if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (0/${totalFiles})...`;
        
        const downloadPromises = objFileInfoList.map(objInfo => {
            const objUrl = downloadUrlMap[objInfo.id];
            if (!objUrl) {
                console.warn(`Could not get download URL for ${objInfo.name}, skipping.`);
                return Promise.resolve(null); // Resolve with null to filter out later
            }
            return fetch(objUrl)
                .then(res => {
                    if (!res.ok) throw new Error(`Failed to download ${objInfo.name} from Box.`);
                    downloadedCount++;
                    if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (${downloadedCount}/${totalFiles})...`;
                    return res.text();
                })
                .then(content => ({ content, info: objInfo }));
        });

        // Wait for all downloads to complete and filter out any that failed
        const allObjContents = (await Promise.all(downloadPromises)).filter(Boolean);
        
        if (loaderTextElement) loaderTextElement.textContent = `Processing geometry...`;

        // --- STEP 5: Parse OBJ header and all geometries ---
        if (allObjContents.length > 0) {
            const headerData = await parseObjHeader(allObjContents[0].content);
            if (headerData) {
                parsedWSCenID = headerData.wscenId;
                parsedPJNo = headerData.pjNo;
            }
        }
        const objLoader = new OBJLoader();
        objLoader.setMaterials(materialsCreator);
        const loadedObjects = allObjContents.map(objData => objLoader.parse(objData.content));

        // --- STEP 6: Combine all parts into a single group, then center and scale it ---
        loadedObjectModelRoot = new THREE.Group();
        loadedObjects.forEach(object => loadedObjectModelRoot.add(...object.children));

        const box = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        const center = box.getCenter(new THREE.Vector3());
        loadedObjectModelRoot.position.sub(center);
        const size = box.getSize(new THREE.Vector3());
        const maxDim = Math.max(size.x, size.y, size.z);
        const scale = (maxDim > 0) ? (150 / maxDim) : 1;
        loadedObjectModelRoot.scale.set(scale, scale, scale);
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;

        // --- STEP 7: Fetch all category data in batches to avoid server input limits ---
        const allIds = [...new Set(
            loadedObjectModelRoot.children
            .map(child => {
                const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                return splitIndex > 0 ? child.name.substring(splitIndex + 1) : null;
            })
            .filter(Boolean) // Filter out nulls
        )];
        await fetchAllCategoryData(parsedWSCenID, allIds);

        // --- STEP 8: Finalize the scene and UI ---
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        updateInfoPanel();

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error);
        if (loaderTextElement) loaderTextElement.textContent = `Error: ${error.message}. Check console for details.`;
    } finally {
        if (loaderContainer) loaderContainer.style.display = 'none';
    }
}


async function parseObjHeader(objContent) {
    try {
        const firstLine = objContent.substring(0, objContent.indexOf('\n')).trim();
        if (firstLine.startsWith("# ")) {
            const content = firstLine.substring(2).trim();
            const match = content.match(/^([0-9a-fA-F-]{36})_(\w+)$/);
            if (match) {
                return { wscenId: match[1], pjNo: match[2] };
            }
            if (content.includes("ワークシェアリングされてない")) {
                return { wscenId: "", pjNo: "" };
            }
        }
    } catch (error) {
        console.error("Error parsing OBJ header:", error);
    }
    return { wscenId: null, pjNo: null };
}

async function fetchAllCategoryData(wscenId, allElementIds) {
    if (!wscenId || allElementIds.length === 0) return;
    
    // This batching is crucial to prevent the "max_input_vars" PHP error.
    const batchSize = 900; 
    
    for (let i = 0; i < allElementIds.length; i += batchSize) {
        const batch = allElementIds.slice(i, i + batchSize);
        if (loaderTextElement) loaderTextElement.textContent = `Fetching Categories... (${i + batch.length}/${allElementIds.length})`;
        
        try {
            const data = await $.ajax({
                type: "post",
                url: url_prefix + "/DLDWH/getDatas",
                data: { _token: CSRF_TOKEN, WSCenID: wscenId, ElementIds: batch }
            });
            for (const elementId in data) {
                elementIdDataMap.set(elementId, data[elementId]);
            }
        } catch (err) {
            console.error(`Error fetching category data batch starting at index ${i}:`, err);
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
    target.traverse(mesh => {
        if (mesh.isMesh && mesh.material) {
            if (!originalMeshMaterials.has(mesh.uuid)) {
                originalMeshMaterials.set(mesh.uuid, mesh.material);
            }
            const materials = Array.isArray(mesh.material) ? mesh.material : [mesh.material];
            const highlightedMaterials = materials.map(originalMat => {
                const highlightMat = originalMat.clone();
                if (highlightMat.color) highlightMat.color.set(color);
                else if (highlightMat.emissive) highlightMat.emissive.set(color); // Fallback for certain materials
                return highlightMat;
            });
            mesh.material = Array.isArray(mesh.material) ? highlightedMaterials : highlightedMaterials[0];
        }
    });
};

const removeAllHighlights = () => {
    originalMeshMaterials.forEach((originalMaterial, meshUuid) => {
        const mesh = scene.getObjectByProperty('uuid', meshUuid);
        if (mesh && mesh.isMesh) {
            mesh.material = originalMaterial;
        }
    });
    originalMeshMaterials.clear();
};

function zoomToAndIsolate(targetObject) {
    if (!targetObject) return;
    deIsolateAllObjects();
    isIsolateModeActive = true;

    // Frame the object
    const box = new THREE.Box3().setFromObject(targetObject);
    if (box.isEmpty()) { isIsolateModeActive = false; return; }
    const center = box.getCenter(new THREE.Vector3());
    const sphere = box.getBoundingSphere(new THREE.Sphere());
    const radius = sphere.radius;
    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    let distance = (radius / Math.sin(fovInRadians / 2)) * 1.5; // Zoom in a bit closer
    const offsetDirection = camera.position.clone().sub(controls.target).normalize();
    camera.position.copy(center).addScaledVector(offsetDirection, distance);
    controls.target.copy(center);
    
    // Isolate effect
    loadedObjectModelRoot.traverse((object) => {
        if (object.isMesh) {
            let isPartOfSelection = false;
            object.traverseAncestors(ancestor => {
                if (ancestor.uuid === targetObject.uuid) isPartOfSelection = true;
            });
            if (object.uuid === targetObject.uuid) isPartOfSelection = true;


            if (!isPartOfSelection) {
                if (!originalObjectPropertiesForIsolate.has(object.uuid)) {
                    originalObjectPropertiesForIsolate.set(object.uuid, { material: object.material, visible: object.visible });
                }
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
    if (!objectInfoPanel) return;
    let headerInfo = `WSCenID: ${parsedWSCenID || "N/A"}\nPJNo: ${parsedPJNo || "N/A"}\n----\n`;
    if (parsedWSCenID === "" && parsedPJNo === "") {
        headerInfo = `WSCenID: \nPJNo: \n----\n`;
    }

    if (selectedObjectOrGroup) {
        let rawName = selectedObjectOrGroup.name || "Unnamed";
        let displayName = rawName;
        let displayId = "N/A";
        const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
        if (splitIndex > 0) {
            displayName = rawName.substring(0, splitIndex);
            displayId = rawName.substring(splitIndex + 1);
        }

        let selectionInfo = `名前: ${displayName}\nID: ${displayId}\n`;
        if (displayId !== "N/A" && elementIdDataMap.has(displayId)) {
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

// --- Event Listeners and Animation Loop ---
window.addEventListener('click', (event) => {
    if (!loadedObjectModelRoot || event.target.closest('#modelTreePanel') || event.target.closest('#model-selector')) return;
    
    const rect = renderer.domElement.getBoundingClientRect();
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);

    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let current = intersects[0].object;
        // Traverse up to find the main group that is a direct child of the root model
        while (current && current.parent !== loadedObjectModelRoot && current.parent !== scene) {
            current = current.parent;
        }
        if (current && current.parent === loadedObjectModelRoot) {
            newlyClickedTarget = current;
        }
    }
    handleSelection(newlyClickedTarget);
});

if (closeModelTreeBtn) closeModelTreeBtn.addEventListener('click', () => { if (modelTreePanel) modelTreePanel.style.display = 'none'; });

if (modelTreeSearch) {
    modelTreeSearch.addEventListener('input', (e) => {
        const searchTerm = e.target.value.toLowerCase().trim();
        const allListItems = modelTreeList.querySelectorAll('li');
        allListItems.forEach(li => {
            const nameSpan = li.querySelector('.group-name');
            const isCategory = li.parentElement === modelTreeList;
            
            if (!nameSpan) return;

            const itemName = nameSpan.textContent.toLowerCase();
            const isMatch = itemName.includes(searchTerm);

            li.style.display = isMatch ? '' : 'none';
            
            if (isMatch && !isCategory) {
                 // If a child matches, ensure its parent category is visible
                let parentLi = li.parentElement.closest('li');
                if (parentLi) {
                    parentLi.style.display = '';
                    const parentSubList = parentLi.querySelector('ul');
                    const parentToggler = parentLi.querySelector('.toggler:not(.empty-toggler)');
                    if (searchTerm) { // Expand categories on search
                         if (parentSubList) parentSubList.style.display = 'block';
                         if (parentToggler) parentToggler.textContent = '▼';
                    }
                }
            }
        });
    });
}

if (toggleUiButton) {
    toggleUiButton.addEventListener('click', () => {
        const isVisible = modelTreePanel.style.display !== 'none';
        if (isVisible) {
            if (modelTreePanel) modelTreePanel.style.display = 'none';
            toggleUiButton.textContent = '📊';
            toggleUiButton.title = "Show UI Panels";
        } else {
            if (modelTreePanel) modelTreePanel.style.display = 'block';
            toggleUiButton.textContent = '❌';
            toggleUiButton.title = "Hide UI Panels";
        }
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
    renderer.autoClearColor = false; // Important for custom background
    renderer.render(scene, camera);
}
