<img width="1902" height="907" alt="image" src="https://github.com/user-attachments/assets/51215e93-f9ad-4a7b-9fbb-c4dbbe503ab4" />
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

// const BOX_ACCESS_TOKEN = $('#access_token').val();
const BOX_MAIN_FOLDER_ID = "332324771912";

// --- Scene Setup, Camera, Renderer, Lighting, Controls ---
const scene = new THREE.Scene();
// scene.background = new THREE.Color(0xcccccc);
// scene.background = new THREE.Color(0xe8e8e8); // A very light gray
// Option B: Gradient Background (More advanced)
// To achieve a gradient, we add a background plane with a custom shader.

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
    // --- Start Application ---
    populateProjectDropdown();
    animate();

});

async function populateProjectDropdown() {
    try {
        const projects = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getProjectList",
                data: { _token: CSRF_TOKEN, folderId: BOX_MAIN_FOLDER_ID },
                success: resolve,
                error: reject
            });
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
            if (loaderTextElement) loaderTextElement.textContent = "No projects found in the main folder.";
        }
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "Error fetching project list from Box.";
    }
}

async function loadModel(projectFolderId, projectName) {
    // 0. Reset scene and state
    if (loadedObjectModelRoot) scene.remove(loadedObjectModelRoot);
    loadedObjectModelRoot = null;
    selectedObjectOrGroup = null;
    originalMeshMaterials.clear();
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
    elementIdDataMap.clear();
    modelTreeList.innerHTML = '';
    if (modelTreePanel) modelTreePanel.style.display = 'none';
    parsedWSCenID = ""; // Reset header info
    parsedPJNo = "";    // Reset header info
    // updateInfoPanel();

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list for ${projectName}...`;

        // 1. Fetch the list of OBJ/MTL file pairs from the server
        const fileList = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getObjList",
                data: { _token: CSRF_TOKEN, folderId: projectFolderId },
                success: resolve,
                error: reject
            });
        });

        if (!fileList || !fileList.mtl || !fileList.objs || fileList.objs.length === 0) {
            throw new Error(`Incomplete file list for project "${projectName}".`);
        }

        const mtlFileInfo = fileList.mtl;
        const objFileInfoList = fileList.objs;

        // 2. Get all file IDs to fetch
        const allFileIds = [mtlFileInfo.id, ...objFileInfoList.map(f => f.id)];

        // 3. Make a SINGLE call to your backend to get all temporary download URLs
        if (loaderTextElement) loaderTextElement.textContent = `Preparing secure downloads...`;
        const downloadUrlMap = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getDownloadUrls",
                data: { _token: window.CSRF_TOKEN, fileIds: allFileIds },
                success: resolve,
                error: reject
            });
        });

        // 4. Load the SINGLE MTL file ONCE directly from Box
        if (loaderTextElement) loaderTextElement.textContent = `Loading Materials...`;
        const mtlUrl = downloadUrlMap[mtlFileInfo.id];
        if (!mtlUrl) throw new Error(`Could not get download URL for MTL file ${mtlFileInfo.name}`);

        const mtlLoader = new MTLLoader();
        const materialsCreator = await mtlLoader.loadAsync(mtlUrl);
        materialsCreator.preload();

        // 5. Download all OBJ file contents in controlled parallel batches, DIRECTLY from Box
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
                if (objInfo && objUrl) {
                    const promise = fetch(objUrl).then(res => {
                        if (!res.ok) throw new Error(`Failed to download ${objInfo.name} from Box: ${res.statusText}`);
                        return res.text();
                    }).then(content => {
                        downloadedCount++;
                        if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (${downloadedCount}/${objFileInfoList.length})...`;
                        return { content, info: objInfo };
                    });
                    currentBatchPromises.push(promise);
                }
            }
            const results = await Promise.all(currentBatchPromises);
            allObjContents.push(...results);
            if (downloadQueue.length > 0) await downloadBatch();
        };

        await downloadBatch();
        // // 2. Load the SINGLE MTL file ONCE
        // if (loaderTextElement) loaderTextElement.textContent = `Loading Materials...`;
        // const mtlContent = await fetchBoxFileContent(mtlFileInfo.id);
        // const mtlLoader = new MTLLoader();
        // const materialsCreator = mtlLoader.parse(mtlContent, '');
        // materialsCreator.preload();

        // // 3. **PERFORMANCE & RATE-LIMIT FIX**: Download all OBJ file contents in controlled parallel batches.
        // if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (0/${objFileInfoList.length})...`;

        // const CONCURRENT_DOWNLOADS = 8;  // Keep this number low (e.g., 4-10)
        // const allObjContents = [];
        // let downloadedCount = 0;

        // // Create a copy of the list to use as a queue
        // const downloadQueue = [...objFileInfoList];

        // const downloadBatch = async () => {
        //     const promises = [];
        //     // Start up to CONCURRENT_DOWNLOADS downloads
        //     while (downloadQueue.length > 0 && promises.length < CONCURRENT_DOWNLOADS) {
        //         const objInfo = downloadQueue.shift(); // Get next file from queue
        //         if (objInfo) {
        //             const promise = fetchBoxFileContent(objInfo.id).then(content => {
        //                 downloadedCount++;
        //                 if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (${downloadedCount}/${objFileInfoList.length})...`;
        //                 return { content: content, info: objInfo };
        //             });
        //             promises.push(promise);
        //         }
        //     }
        //     // Wait for this batch of promises to complete
        //     const results = await Promise.all(promises);
        //     allObjContents.push(...results); // Add results to the main array

        //     // If there are more files in the queue, recurse to download the next batch
        //     if (downloadQueue.length > 0) {
        //         await downloadBatch();
        //     }
        // };

        // await downloadBatch(); // Start the first batch

        // 4. Parse the header from the FIRST downloaded OBJ file
        if (allObjContents.length > 0) {
            const headerData = await parseObjHeader(allObjContents[0].content);
            if (headerData) {
                parsedWSCenID = headerData.wscenId;
                parsedPJNo = headerData.pjNo;
            }
        }

        // 5. Parse all OBJ geometries using the SAME material creator
        if (loaderTextElement) loaderTextElement.textContent = `Processing geometry...`;
        const objLoader = new OBJLoader();
        objLoader.setMaterials(materialsCreator);

        const loadedObjects = allObjContents.map(objData => {
            return objLoader.parse(objData.content);
        });

        // 6. Combine all loaded objects into a single group
        const combinedModel = new THREE.Group();
        loadedObjects.forEach(object => {
            while (object.children.length > 0) {
                combinedModel.add(object.children[0]);
            }
        });
        loadedObjectModelRoot = combinedModel;

        // 7. Process the COMBINED model (center, scale, rotate)
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

        // 8. Fetch category data for all parts
        const allIds = [];
        loadedObjectModelRoot.traverse(child => {
            if (child.name) {
                const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                if (splitIndex > 0) allIds.push(child.name.substring(splitIndex + 1));
            }
        });
        await fetchAllCategoryData(parsedWSCenID, [...new Set(allIds)]);

        // 9. Build the tree, add to scene, frame, and hide loader
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        if (loaderContainer) loaderContainer.style.display = 'none';
        updateInfoPanel();

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model: ${projectName}. Check console.`;
        if (loaderContainer) loaderContainer.style.display = 'none';
    }
}

// This function now acts as a simple wrapper for your backend endpoint
async function fetchBoxFileContent(fileId) {
    // console.log("Fetching content for File ID:", fileId);
    try {
        const fileContent = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getFileContents", // Your new backend route
                data: { _token: CSRF_TOKEN, fileId: fileId },
                success: resolve, // On success, resolve the promise with the raw text data
                error: (jqXHR, textStatus, errorThrown) => {
                    console.error(`AJAX error for file ID ${fileId}: ${textStatus} - ${errorThrown}`);
                    if (loaderContainer) loaderContainer.style.display = 'none';

                    // Reject with a meaningful error
                    reject(new Error(`AJAX error for file ID ${fileId}: ${jqXHR.responseJSON ? JSON.stringify(jqXHR.responseJSON) : errorThrown}`));
                }
            });
        });

        // The promise resolves with the file content directly.
        // We can add a simple check to see if we got something back.
        if (typeof fileContent !== 'string' || fileContent.length === 0) {
            throw new Error(`Received empty or invalid content for file ${fileId}`);
        }
        return fileContent; // Return the raw text content
    } catch (error) {
        console.error(`Failed inside fetchBoxFileContent for file ID ${fileId}:`, error);
        // Re-throw the error so Promise.all in the main loader function can catch it
        throw error;
    }
}


async function parseObjHeader(objContent) {
    // This function now only parses and returns the values, not setting globals.
    try {
        const lines = objContent.split(/\r?\n/);
        if (lines.length > 0) {
            const firstLine = lines[0].trim();
            if (firstLine.startsWith("# ")) {
                const content = firstLine.substring(2).trim();
                const pattern1Match = content.match(/^([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
                if (pattern1Match) {
                    return { wscenId: pattern1Match[1], pjNo: pattern1Match[2] };
                }
                if (content.includes("ワークシェアリングされてない")) {
                    return { wscenId: "", pjNo: "" };
                }
            }
        }
    } catch (error) {
        console.error("Error parsing OBJ header:", error);
    }
    return { wscenId: null, pjNo: null }; // Return nulls if no match
}

async function fetchAllCategoryData(wscenId, allElementIds) {
    if (allElementIds.length === 0) return;
    const batchSize = 900;
    for (let i = 0; i < allElementIds.length; i += batchSize) {
        const batch = allElementIds.slice(i, i + batchSize);
        if (loaderTextElement) loaderTextElement.textContent = `Fetching Categories... (${i + batch.length}/${allElementIds.length})`;
        await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/DLDWH/getDatas",
                data: { _token: CSRF_TOKEN, WSCenID: wscenId, ElementIds: batch },
                success: (data) => {
                    for (const elementId in data) {
                        elementIdDataMap.set(elementId, data[elementId]);
                    }
                    resolve();
                },
                error: (err) => {
                    console.error(`Error fetching batch starting at index ${i}:`, err);
                    reject(err);
                }
            });
        });
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

// --- Event Listeners and Animation Loop ---
window.addEventListener('click', (event) => {
    if (!loadedObjectModelRoot || event.target.closest('#modelTreePanel')) return;
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
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
});

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

// window.addEventListener('resize', () => {
//     camera.aspect = window.innerWidth / window.innerHeight;
//     camera.updateProjectionMatrix();
//     renderer.setSize(window.innerWidth, window.innerHeight);
// });

// This function now correctly resizes the renderer based on its container's dimensions
function onWindowResize() {
    const { clientWidth, clientHeight } = viewerContainer;

    camera.aspect = clientWidth / clientHeight;
    camera.updateProjectionMatrix();

    renderer.setSize(clientWidth, clientHeight);
}
window.addEventListener('resize', onWindowResize);

// --- Event Listeners and Animation Loop ---
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
