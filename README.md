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
const resetViewButton = document.getElementById('resetViewButton'); // Added

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
let mouseDownPosition = new THREE.Vector2(); // Added for click vs drag detection
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

    // Added: Event listener for the Reset View button
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
            alert("BOXにログインされていないためobjファイルを取得きませんでした。");
            if (loaderTextElement) loaderTextElement.textContent = "Box login required.";
        } else if (projects && projects.length > 0) {
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
    resetScene();
    if (loaderContainer) loaderContainer.style.display = 'flex';

    try {
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list for ${projectName}...`;
        const filePairs = await $.ajax({
            type: "post",
            url: url_prefix + "/box/getObjList",
            data: { _token: CSRF_TOKEN, folderId: projectFolderId }
        });
        if (!Array.isArray(filePairs) || filePairs.length === 0) {
            throw new Error(`No valid OBJ files found for project "${projectName}".`);
        }

        const allFilesToFetch = [];
        filePairs.forEach(pair => {
            if (pair.obj) allFilesToFetch.push(pair.obj);
            if (pair.mtl) allFilesToFetch.push(pair.mtl);
        });
        const allFileIds = [...new Set(allFilesToFetch.map(f => f.id))];
        const downloadUrlMap = await fetchAllDownloadUrlsInBatches(allFileIds);

        const allFileContents = await downloadAllFilesWithConcurrency(allFilesToFetch, downloadUrlMap);

        if (loaderTextElement) loaderTextElement.textContent = `Processing geometry and materials...`;

        const allLoadedObjects = [];
        const contentMap = new Map(allFileContents.map(item => [item.info.id, item.content]));

        const firstObjContent = allFileContents.find(f => f.info.name.toLowerCase().endsWith('.obj'))?.content;
        if (firstObjContent) {
            const headerData = await parseObjHeader(firstObjContent);
            if (headerData) {
                parsedWSCenID = headerData.wscenId;
                parsedPJNo = headerData.pjNo;
            }
        }

        for (const pair of filePairs) {
            try {
                const objContent = contentMap.get(pair.obj.id);
                if (!objContent || !objContent.trim()) {
                    console.warn(`Skipping empty or missing OBJ file: ${pair.obj.name}`);
                    continue;
                }

                let materialsCreator = null;
                if (pair.mtl) {
                    const mtlContent = contentMap.get(pair.mtl.id);
                    if (mtlContent && mtlContent.trim()) {
                        const mtlLoader = new MTLLoader();
                        materialsCreator = mtlLoader.parse(mtlContent, '');
                    }
                }

                const objLoader = new OBJLoader();
                if (materialsCreator) {
                    objLoader.setMaterials(materialsCreator);
                }

                const parsedGroup = objLoader.parse(objContent);
                if (parsedGroup) {
                    allLoadedObjects.push(parsedGroup);
                }

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
        
        // Added: Store scale factor for volume calculation
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

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error);
        if (loaderTextElement) loaderTextElement.textContent = `Error: ${error.message}. Check console.`;
    } finally {
        if (loaderContainer) loaderContainer.style.display = 'none';
    }
}

async function downloadAllFilesWithConcurrency(fileInfoList, downloadUrlMap, concurrencyLimit = 20) {
    const queue = [...fileInfoList];
    const allResults = [];
    let downloadedCount = 0;
    const totalFiles = fileInfoList.length;

    const worker = async () => {
        while (queue.length > 0) {
            const fileInfo = queue.shift();
            if (!fileInfo) continue;
            const fileUrl = downloadUrlMap[fileInfo.id];
            if (!fileUrl) {
                console.warn(`Could not get download URL for ${fileInfo.name}, skipping.`);
                continue;
            }
            try {
                const response = await fetch(fileUrl);
                if (!response.ok) throw new Error(`HTTP error! Status: ${response.status}`);
                const content = await response.text();
                allResults.push({ content, info: fileInfo });
                downloadedCount++;
                if (loaderTextElement) {
                    loaderTextElement.textContent = `Downloading Files (${downloadedCount}/${totalFiles})...`;
                }
            } catch (error) {
                console.error(`Failed to download ${fileInfo.name}:`, error);
            }
        }
    };
    const workerPromises = [];
    for (let i = 0; i < concurrencyLimit; i++) {
        workerPromises.push(worker());
    }
    await Promise.all(workerPromises);
    return allResults;
}

function resetScene() {
    if (loadedObjectModelRoot) {
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

async function fetchAllDownloadUrlsInBatches(allFileIds) {
    const allUrls = {};
    const batchSize = 250;
    for (let i = 0; i < allFileIds.length; i += batchSize) {
        const batch = allFileIds.slice(i, i + batchSize);
        if (loaderTextElement) {
            loaderTextElement.textContent = `Preparing secure downloads... (${i + batch.length}/${allFileIds.length})`;
        }
        try {
            const batchUrls = await $.ajax({
                type: "post",
                url: url_prefix + "/box/getDownloadUrls",
                data: { _token: CSRF_TOKEN, fileIds: batch }
            });
            Object.assign(allUrls, batchUrls);
        } catch (err) {
            console.error(`Error fetching download URL batch starting at index ${i}:`, err);
        }
    }
    return allUrls;
}

async function parseObjHeader(objContent) {
    try {
        const firstLine = objContent.substring(0, objContent.indexOf('\n')).trim();
        if (firstLine.startsWith("# ")) {
            const content = firstLine.substring(2).trim();
            const match = content.match(/^([0-9a-fA-F-]{36})_(\w+)$/);
            if (match) { return { wscenId: match[1], pjNo: match[2] }; }
            if (content.includes("ワークシェアリングされてない")) { return { wscenId: "", pjNo: "" }; }
        }
    } catch (error) { console.error("Error parsing OBJ header:", error); }
    return { wscenId: null, pjNo: null };
}

async function fetchAllCategoryData(wscenId, allElementIds) {
    if (!wscenId || allElementIds.length === 0) return;
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
            for (const elementId in data) { elementIdDataMap.set(elementId, data[elementId]); }
        } catch (err) { console.error(`Error fetching category data batch starting at index ${i}:`, err); }
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
            } else if (!displayId) { category = "名称未分類"; }
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
        console.warn("Cannot frame object with empty bounding box.");
        camera.position.set(50, 50, 50);
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
    
    // Modified: Start with the tree expanded
    toggler.textContent = '▼';
    
    itemContent.appendChild(toggler);
    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = `${categoryName} (${objectsInCategory.length})`;
    itemContent.appendChild(nameSpan);
    const subList = document.createElement('ul');
    
    // Modified: Start with the tree expanded
    subList.style.display = 'block';

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

// Modified: More efficient highlight function
const applyHighlight = (target, color) => {
    if (!target) return;
    // Use a single, shared material for highlighting for better performance
    const highlightMaterial = new THREE.MeshBasicMaterial({
        color: color,
        side: THREE.DoubleSide
    });

    target.traverse(child => {
        if (child.isMesh) {
            if (!originalMeshMaterials.has(child.uuid)) {
                originalMeshMaterials.set(child.uuid, child.material);
            }
            child.material = highlightMaterial;
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
    
    // Isolation logic remains the same
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

// Modified: Includes Volume Calculation
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

        // Added: Calculate and display volume
        const volumeInOriginalUnits = getVolumeOfSelectedObject();
        if (volumeInOriginalUnits > 0.0001) {
            // Assuming original units are mm, convert mm^3 to m^3
            const CUBIC_MM_PER_CUBIC_M = 1e9; // 1000*1000*1000
            const volumeInM3 = volumeInOriginalUnits / CUBIC_MM_PER_CUBIC_M;
            selectionInfo += `\n体積: ${volumeInM3.toFixed(5)} m³`;
        }

        objectInfoPanel.textContent = headerInfo + selectionInfo;
    } else {
        objectInfoPanel.textContent = headerInfo + 'None Selected';
    }
}

// Modified: Listener attached to viewerContainer, not window
viewerContainer.addEventListener('mousedown', (event) => {
    mouseDownPosition.set(event.clientX, event.clientY);
}, false);

// Modified: Listener attached to viewerContainer, not window
viewerContainer.addEventListener('mouseup', (event) => {
    const mouseUpPosition = new THREE.Vector2(event.clientX, event.clientY);
    const DRAG_THRESHOLD = 5;
    if (mouseDownPosition.distanceTo(mouseUpPosition) > DRAG_THRESHOLD) return;

    if (!loadedObjectModelRoot) return;
    const rect = viewerContainer.getBoundingClientRect();
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);
    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let current = intersects[0].object;
        while (current && current.parent !== loadedObjectModelRoot && current.parent !== scene) {
            current = current.parent;
        }
        if (current && current.parent === loadedObjectModelRoot) {
            newlyClickedTarget = current;
        }
    }
    handleSelection(newlyClickedTarget);
}, false);

if (closeModelTreeBtn) closeModelTreeBtn.addEventListener('click', () => { if (modelTreePanel) modelTreePanel.style.display = 'none'; });

if (modelTreeSearch) {
    modelTreeSearch.addEventListener('input', (e) => {
        const searchTerm = e.target.value.toLowerCase().trim();
        document.querySelectorAll('#modelTreeList li').forEach(li => {
            const nameSpan = li.querySelector('.group-name');
            if (!nameSpan) return;
            const isMatch = nameSpan.textContent.toLowerCase().includes(searchTerm);
            li.style.display = isMatch ? '' : 'none';
            if (isMatch) {
                let parent = li.parentElement.closest('li');
                while (parent) {
                    parent.style.display = '';
                    if (searchTerm) {
                        parent.querySelector('ul').style.display = 'block';
                        parent.querySelector('.toggler').textContent = '▼';
                    }
                    parent = parent.parentElement.closest('li');
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
    renderer.setPixelRatio(window.devicePixelRatio);
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

// Added: New reset view function
function handleResetView() {
    deIsolateAllObjects();
    removeAllHighlights();
    selectedObjectOrGroup = null;
    document.querySelectorAll('#modelTreePanel .tree-item.selected').forEach(el => el.classList.remove('selected'));
    updateInfoPanel();
    // NOTE: Does NOT call frameObject, so the camera view is preserved.
}

// Added: Volume calculation function
function getVolumeOfSelectedObject() {
    if (!selectedObjectOrGroup || !loadedObjectModelRoot) {
        return 0;
    }
    let totalVolume = 0;
    selectedObjectOrGroup.traverse(child => {
        if (child.isMesh && child.geometry) {
            totalVolume += calculateMeshVolume(child);
        }
    });

    const modelScale = loadedObjectModelRoot.userData.modelScale || 1;
    if (modelScale === 0) return 0;
    
    // Correct for the scaling applied during load to get the original volume
    const correctedVolume = totalVolume / Math.pow(modelScale, 3);
    return correctedVolume;
}

// Added: Helper function to calculate a single mesh's volume
function calculateMeshVolume(mesh) {
    const geometry = mesh.geometry;
    if (!geometry.isBufferGeometry) return 0;

    let totalVolume = 0;
    const position = geometry.attributes.position;
    const a = new THREE.Vector3();
    const b = new THREE.Vector3();
    const c = new THREE.Vector3();

    const processTriangle = (vA_idx, vB_idx, vC_idx) => {
        a.fromBufferAttribute(position, vA_idx);
        b.fromBufferAttribute(position, vB_idx);
        c.fromBufferAttribute(position, vC_idx);

        a.applyMatrix4(mesh.matrixWorld);
        b.applyMatrix4(mesh.matrixWorld);
        c.applyMatrix4(mesh.matrixWorld);

        totalVolume += a.dot(b.clone().cross(c));
    };

    if (geometry.index) {
        const index = geometry.index;
        for (let i = 0; i < index.count; i += 3) {
            processTriangle(index.getX(i), index.getX(i + 1), index.getX(i + 2));
        }
    } else {
        for (let i = 0; i < position.count; i += 3) {
            processTriangle(i, i + 1, i + 2);
        }
    }
    return Math.abs(totalVolume / 6.0);
}
